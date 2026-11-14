// Mente Digital · POST /api/pesquisar
// Fonte: OpenStreetMap (Nominatim + Overpass). Gratuita, sem chave de API.
// Nenhum resultado é inventado: tudo vem da fonte ou aparece como "Não encontrado".

const UA = `MenteDigital/1.0 (${process.env.CONTACT_EMAIL || 'defina-CONTACT_EMAIL'})`;

const NICHOS = [
  { n: 'Restaurantes e hamburguerias', q: ['["amenity"~"^(restaurant|fast_food)$"]'], t: t => /^(restaurant|fast_food)$/.test(t.amenity) },
  { n: 'Barbearias e salões de beleza', q: ['["shop"~"^(hairdresser|beauty)$"]'], t: t => /^(hairdresser|beauty)$/.test(t.shop) },
  { n: 'Clínicas e consultórios', q: ['["amenity"~"^(clinic|doctors)$"]'], t: t => /^(clinic|doctors)$/.test(t.amenity) },
  { n: 'Dentistas', q: ['["amenity"="dentist"]'], t: t => t.amenity === 'dentist' },
  { n: 'Imobiliárias', q: ['["office"="estate_agent"]'], t: t => t.office === 'estate_agent' },
  { n: 'Academias', q: ['["leisure"="fitness_centre"]'], t: t => t.leisure === 'fitness_centre' },
  { n: 'Oficinas mecânicas', q: ['["shop"="car_repair"]'], t: t => t.shop === 'car_repair' },
  { n: 'Assistências técnicas', q: ['["craft"="electronics_repair"]', '["shop"~"^(mobile_phone|computer)$"]'], t: t => t.craft === 'electronics_repair' || /^(mobile_phone|computer)$/.test(t.shop) },
  { n: 'Escritórios de advocacia', q: ['["office"="lawyer"]'], t: t => t.office === 'lawyer' },
  { n: 'Lojas', q: ['["shop"]'], t: t => !!t.shop },
  { n: 'Prestadores de serviços', q: ['["office"]'], t: t => !!t.office },
  { n: 'Outros segmentos', q: ['["craft"]'], t: t => !!t.craft },
];

// Perfis sociais e agregadores NÃO contam como site oficial.
const SOCIAL = /(instagram|facebook|fb\.com|linktr\.ee|ifood|wa\.me|whatsapp|tripadvisor|google\.[a-z.]+\/maps|youtube|tiktok|twitter|x\.com)/i;
const PRIVATE_HOST = /^(localhost|127\.|10\.|192\.168\.|169\.254\.|172\.(1[6-9]|2\d|3[01])\.|0\.|\[)/i;

const hits = new Map(); // limite simples por IP (melhor esforço em ambiente serverless)
function limited(ip) {
  const now = Date.now();
  const a = (hits.get(ip) || []).filter(t => now - t < 36e5);
  if (a.length >= 10) { hits.set(ip, a); return true; }
  a.push(now); hits.set(ip, a); return false;
}

const sleep = ms => new Promise(r => setTimeout(r, ms));

async function getJson(url, opts = {}, ms = 25000) {
  const c = new AbortController();
  const timer = setTimeout(() => c.abort(), ms);
  try {
    const r = await fetch(url, { ...opts, signal: c.signal, headers: { 'User-Agent': UA, ...(opts.headers || {}) } });
    if (!r.ok) throw new Error(`HTTP ${r.status} em ${new URL(url).host}`);
    return await r.json();
  } finally { clearTimeout(timer); }
}

async function areaId(q) {
  const r = await getJson('https://nominatim.openstreetmap.org/search?format=jsonv2&limit=5&q=' + encodeURIComponent(q));
  const rel = r.find(x => x.osm_type === 'relation');
  return rel ? 3600000000 + Number(rel.osm_id) : null;
}

function overpass(id, sels) {
  const body = sels.map(s => `nwr${s}["name"](area.a);`).join('');
  const q = `[out:json][timeout:25];area(${id})->.a;(${body});out center tags 400;`;
  return getJson('https://overpass-api.de/api/interpreter', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: 'data=' + encodeURIComponent(q),
  }, 30000);
}

function normalizar(el, local, agora) {
  const t = el.tags || {};
  const lat = el.lat ?? el.center?.lat, lon = el.lon ?? el.center?.lon;
  let site = t.website || t['contact:website'] || t.url || null;
  let redes = null;
  if (site && SOCIAL.test(site)) { redes = site; site = null; }
  const rua = [t['addr:street'], t['addr:housenumber']].filter(Boolean).join(', ');
  const endereco = [rua, t['addr:suburb']].filter(Boolean).join(' - ') || null;
  return {
    nome: t.name,
    nicho: (NICHOS.find(n => n.t(t)) || { n: 'Outros segmentos' }).n,
    endereco,
    cidade: t['addr:city'] || local,
    telefone: t.phone || t['contact:phone'] || null,
    whatsapp: t['contact:whatsapp'] || null,
    instagram: t['contact:instagram'] || t.instagram || (redes && /instagram/i.test(redes) ? redes : null),
    site,
    status: null,
    evidencia: null,
    fonte: 'OpenStreetMap',
    fonteUrl: `https://www.openstreetmap.org/${el.type}/${el.id}`,
    pesquisadoEm: agora,
    _lat: lat, _lon: lon,
  };
}

const chave = l => {
  const nome = l.nome.toLowerCase().normalize('NFD').replace(/[\u0300-\u036f]/g, '').replace(/[^a-z0-9]/g, '');
  const tel = (l.telefone || '').replace(/\D/g, '').slice(-8);
  const geo = l._lat != null ? `${l._lat.toFixed(3)},${l._lon.toFixed(3)}` : '';
  return tel ? `${nome}|${tel}` : `${nome}|${geo}`;
};

async function ping(u) {
  if (!/^https?:\/\//i.test(u)) u = 'http://' + u;
  let host;
  try { host = new URL(u).hostname; } catch { return null; }
  if (PRIVATE_HOST.test(host)) return null;
  const c = new AbortController();
  const timer = setTimeout(() => c.abort(), 5000);
  try {
    const r = await fetch(u, { redirect: 'follow', signal: c.signal, headers: { 'User-Agent': UA } });
    return r.status < 500;
  } catch { return false; } finally { clearTimeout(timer); }
}

async function verificar(l) {
  if (!l.site) {
    // Sem site na fonte: NUNCA confirma ausência automaticamente.
    l.status = 'Site não localizado';
    l.evidencia = 'Nenhum site registrado na fonte. Ausência não confirmada.';
    return;
  }
  const ok = await ping(l.site);
  if (ok) { l.status = 'Site encontrado'; l.evidencia = 'Site informado pela fonte e respondeu à verificação.'; }
  else { l.status = 'Verificação pendente'; l.evidencia = 'Site informado pela fonte, mas não foi possível confirmar que está no ar.'; }
}

module.exports = async (req, res) => {
  res.setHeader('Cache-Control', 'no-store');
  if (req.method !== 'POST') return res.status(405).json({ erro: 'Use POST.' });
  const ip = String(req.headers['x-forwarded-for'] || '').split(',')[0].trim() || 'desconhecido';
  if (limited(ip)) return res.status(429).json({ erro: 'Limite de pesquisas atingido. Tente novamente mais tarde.' });

  let b;
  try { b = typeof req.body === 'string' ? JSON.parse(req.body || '{}') : (req.body || {}); }
  catch { return res.status(400).json({ erro: 'Corpo da requisição inválido.' }); }

  const str = v => (typeof v === 'string' ? v.trim().slice(0, 80) : '');
  const pais = str(b.pais), estado = str(b.estado);
  const cidades = (Array.isArray(b.cidades) ? b.cidades : []).map(str).filter(Boolean).slice(0, 5);
  const escolhidos = NICHOS.filter(n => (Array.isArray(b.nichos) ? b.nichos : []).includes(n.n));
  const quantidade = Math.min(Math.max(parseInt(b.quantidade, 10) || 50, 1), 100);

  if (!escolhidos.length) return res.status(400).json({ erro: 'Selecione ao menos um nicho válido.' });
  if (!cidades.length && !estado) return res.status(400).json({ erro: 'Informe ao menos uma cidade ou um estado. Pesquisa por país inteiro não é suportada.' });

  const locais = cidades.length ? cidades.map(c => [c, estado, pais].filter(Boolean).join(', ')) : [[estado, pais].filter(Boolean).join(', ')];
  const sels = escolhidos.flatMap(n => n.q);
  const agora = new Date().toISOString();
  const erros = [];
  let todos = [];

  for (let i = 0; i < locais.length; i++) {
    if (i) await sleep(1100); // política do Nominatim: 1 requisição por segundo
    try {
      const id = await areaId(locais[i]);
      if (!id) { erros.push(`Área não encontrada como limite administrativo: ${locais[i]}`); continue; }
      const r = await overpass(id, sels);
      todos = todos.concat((r.elements || []).filter(e => e.tags && e.tags.name).map(e => normalizar(e, locais[i], agora)));
    } catch (e) { erros.push(`${locais[i]}: ${e.name === 'AbortError' ? 'tempo esgotado na fonte' : e.message}`); }
  }

  if (!todos.length && erros.length) return res.status(502).json({ erro: 'Falha ao consultar a fonte. ' + erros.join(' | ') });

  const vistos = new Set();
  let leads = todos.filter(l => { const k = chave(l); if (vistos.has(k)) return false; vistos.add(k); return true; });
  if (b.priorizarSemSite) leads.sort((a, c) => !!a.site - !!c.site);
  leads = leads.slice(0, quantidade);

  for (let i = 0; i < leads.length; i += 8) await Promise.all(leads.slice(i, i + 8).map(verificar));

  leads.forEach(l => { delete l._lat; delete l._lon; });
  res.status(200).json({ leads, total: leads.length, fonte: 'OpenStreetMap (Nominatim + Overpass)', pesquisadoEm: agora, erros });
};
