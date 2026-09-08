# hospital-ramonexeni-pages

Wrapper (GitHub Pages) que sirve el **backoffice WA del Hospital del Niño Ramón Exeni**
bajo `hospitalramonexeni.giwa-ia.com`, enmascarando que es Apps Script.

- `index.html`: login con Google Identity Services (GIS) → carga el `/exec` del
  backoffice en un iframe a pantalla completa, pasando el ID token por `?t=<JWT>`.
- El backoffice valida el token server-side y autoriza contra `wa.usuarios`.
- `CNAME`: hospitalramonexeni.giwa-ia.com

Config a completar en `index.html`:
- `CLIENT_ID`: OAuth Client ID (GIS) del subdominio (Web app, origin https://hospitalramonexeni.giwa-ia.com).
- `APP_EXEC`: URL /exec del deployment estable del backoffice (con ALLOWALL y acceso "Anyone").

Mismo patrón que app-giwa / crm-giwa. DNS: CNAME `hospitalramonexeni` → sechavarria-star.github.io (Squarespace).
