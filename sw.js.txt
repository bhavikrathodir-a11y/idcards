/* Keeps the app shell (this small page) available; the systems themselves always load live from Google. */
const CACHE = 'idcards-shell-v12';
const SHELL = ['./', './index.html', './manifest.json', './icon-192.png', './icon-512.png'];
self.addEventListener('install', e => {
  e.waitUntil(caches.open(CACHE)
    .then(c => Promise.all(SHELL.map(u => c.add(u).catch(() => { }))))   /* one missing file must not stop the install */
    .then(() => self.skipWaiting()));
});
self.addEventListener('activate', e => { e.waitUntil(caches.keys().then(ks => Promise.all(ks.filter(k => k !== CACHE).map(k => caches.delete(k)))).then(() => self.clients.claim())); });
self.addEventListener('fetch', e => {
  const u = new URL(e.request.url);
  if (e.request.method !== 'GET' || u.origin !== location.origin) return;       /* Google pages: never cached */
  if (u.pathname.indexOf('config.js') >= 0) return;                             /* links file: always straight from the network */
  e.respondWith(fetch(e.request, { cache: 'no-store' })
    .then(r => {
      if (r && r.ok && r.type === 'basic') {                                    /* never keep a 404 or an error page */
        const cp = r.clone();
        caches.open(CACHE).then(c => c.put(e.request, cp));
        return r;
      }
      return caches.match(e.request, { ignoreSearch: true }).then(hit => hit || r);
    })
    .catch(() => caches.match(e.request, { ignoreSearch: true }).then(r => r || caches.match('./index.html'))));
});
