# AfricaMoja

## One Africa — 1‑Day MVP Scope & Sprint Plan (Angular + Firebase)

> Goal: Ship a public, multilingual Angular site with a clickable Africa map, radial wheel, basic project browsing (from JSON), and Firebase Hosting deployment — **in one day**.

---

### 0) Hard Scope Cut (Day‑1)

**Included**

* Landing page: header + short explainer + carousel (Swiper)
* Clickable Africa map (Leaflet) → filters project list by country code
* Radial wheel (SVG) with 6 axes → filters by axis (and shows sub‑icons on hover)
* Project list + client-side filters (axis, country) from **static JSON** (no Firestore today)
* Project details route pulling from same JSON
* Multilingual (EN + FR) via ngx-translate
* Firebase Hosting deploy (prod-ready URL)

**Deferred** (post‑day‑1)

* Auth, submissions, votes/comments, Firestore write rules, admin, search, offline/PWA polish

---

### 1) Timeline (suggested)

* **Hour 1:** Repo + Angular scaffold + Tailwind + i18n + assets folders
* **Hour 2:** Landing header + carousel
* **Hour 3:** Radial wheel (SVG) component + axis filter wiring
* **Hour 4:** Africa map (Leaflet) + country filters
* **Hour 5:** Project list (cards) + detail route (Markdown-ish steps)
* **Hour 6:** i18n pass (EN/FR), basic accessibility, mobile tweaks
* **Hour 7:** Firebase Hosting setup + deploy + smoke tests
* **Hour 8:** Buffer: fix bugs, polish, hero copy, add 6–10 seed projects

---

### 2) Commands

```bash
# Create Angular app
ng new one-africa --style=css --routing --standalone
cd one-africa

# Deps
npm i leaflet swiper ngx-translate-core ngx-translate-http-loader marked
npm i -D tailwindcss postcss autoprefixer

# Tailwind init
npx tailwindcss init -p

# Firebase Hosting
npm i -g firebase-tools
firebase login
firebase init hosting
```

**tailwind.config.js**

```js
module.exports = {
  content: ["./src/**/*.{html,ts}"],
  theme: { extend: {} },
  plugins: [],
};
```

**src/styles.css**

```css
@tailwind base; @tailwind components; @tailwind utilities;
/* Leaflet CSS */
@import url('https://unpkg.com/leaflet@1.9.4/dist/leaflet.css');
```

---

### 3) Project Structure (added files)

```
src/assets/
  geo/africa.geojson
  i18n/en.json
  i18n/fr.json
  data/projects.json
  icons/*.svg
src/app/
  app.routes.ts
  core/i18n/translate.provider.ts
  shared/components/radial-wheel/radial-wheel.component.ts
  features/landing/landing.component.ts
  features/map/map.component.ts
  features/projects/list/projects-list.component.ts
  features/projects/detail/project-detail.component.ts
```

---

### 4) i18n Setup (ngx-translate)

**src/app/core/i18n/translate.provider.ts**

```ts
import { HttpClient } from '@angular/common/http';
import { importProvidersFrom } from '@angular/core';
import { TranslateLoader, TranslateModule } from 'ngx-translate-core';
import { TranslateHttpLoader } from 'ngx-translate-http-loader';

export function HttpLoaderFactory(http: HttpClient) {
  return new TranslateHttpLoader(http, './assets/i18n/', '.json');
}

export const provideI18n = () => [
  importProvidersFrom(TranslateModule.forRoot({
    loader: { provide: TranslateLoader, useFactory: HttpLoaderFactory, deps: [HttpClient] },
    defaultLanguage: 'en'
  }))
];
```

**src/assets/i18n/en.json**

```json
{
  "title": "One Africa",
  "tagline": "Local wisdom, shared solutions.",
  "browse": "Browse projects",
  "axes": { "water": "Water", "food": "Food", "organizing": "Organizing", "sciences": "Sciences", "avoidances": "Avoidances", "clothing": "Clothing", "crafts": "Crafts" }
}
```

**src/assets/i18n/fr.json**

```json
{
  "title": "One Africa",
  "tagline": "Savoirs locaux, solutions partagées.",
  "browse": "Parcourir les projets",
  "axes": { "water": "Eau", "food": "Alimentation", "organizing": "Organisation", "sciences": "Sciences", "avoidances": "Évitements", "clothing": "Habillement", "crafts": "Artisanat" }
}
```

---

### 5) Routing

**src/app/app.routes.ts**

```ts
import { Routes } from '@angular/router';
export const routes: Routes = [
  { path: '', loadComponent: () => import('./features/landing/landing.component').then(m=>m.LandingComponent) },
  { path: 'map', loadComponent: () => import('./features/map/map.component').then(m=>m.MapComponent) },
  { path: 'projects', loadComponent: () => import('./features/projects/list/projects-list.component').then(m=>m.ProjectsListComponent) },
  { path: 'projects/:slug', loadComponent: () => import('./features/projects/detail/project-detail.component').then(m=>m.ProjectDetailComponent) },
  { path: '**', redirectTo: '' }
];
```

---

### 6) Radial Wheel (SVG)

**src/app/shared/components/radial-wheel/radial-wheel.component.ts**

```ts
import { Component, EventEmitter, Output } from '@angular/core';
const AXES = ['water','food','organizing','sciences','avoidances','clothing','crafts'];

@Component({
  selector: 'oa-radial-wheel',
  standalone: true,
  template: `
  <div class="w-full flex items-center justify-center">
    <svg viewBox="0 0 200 200" class="max-w-md">
      <ng-container *ngFor="let a of axes; let i = index">
        <path [attr.d]="arc(i, axes.length)" class="cursor-pointer hover:opacity-80"
              (click)="select(a)" fill="currentColor"/>
        <text [attr.x]="labelPos(i, axes.length).x" [attr.y]="labelPos(i, axes.length).y"
              text-anchor="middle" alignment-baseline="middle" class="text-xs">{{ 'axes.'+a | translate }}</text>
      </ng-container>
      <circle cx="100" cy="100" r="28" fill="white" stroke="#ddd"/>
    </svg>
  </div>`,
  styles: [":host{display:block}"]
})
export class RadialWheelComponent {
  axes = AXES;
  @Output() axisSelected = new EventEmitter<string>();
  select(a:string){ this.axisSelected.emit(a); }
  arc(i:number,n:number){
    const start = (2*Math.PI*i)/n, end = (2*Math.PI*(i+1))/n;
    const r = 90, cx=100, cy=100;
    const x1=cx + r*Math.cos(start), y1=cy + r*Math.sin(start);
    const x2=cx + r*Math.cos(end),   y2=cy + r*Math.sin(end);
    return `M ${cx} ${cy} L ${x1} ${y1} A ${r} ${r} 0 0 1 ${x2} ${y2} Z`;
  }
  labelPos(i:number,n:number){
    const ang = (2*Math.PI*(i+0.5))/n; const r=65, cx=100, cy=100;
    return { x: cx + r*Math.cos(ang), y: cy + r*Math.sin(ang) };
  }
}
```

---

### 7) Africa Map (Leaflet)

**src/app/features/map/map.component.ts**

```ts
import { Component, AfterViewInit } from '@angular/core';
import * as L from 'leaflet';
@Component({ selector:'oa-map', standalone:true, template:`<div id="map" class="h-[60vh] rounded-2xl shadow"/>` })
export class MapComponent implements AfterViewInit {
  ngAfterViewInit(){
    const map = L.map('map', { zoomControl: true }).setView([1.5, 17.3], 3);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { maxZoom: 8 }).addTo(map);
    fetch('/assets/geo/africa.geojson').then(r=>r.json()).then((geo)=>{
      const layer = L.geoJSON(geo, {
        style: { weight:1, color:'#888', fillColor:'#ddd', fillOpacity:0.6 },
        onEachFeature: (f:any, l:any) => l.on('click', ()=>{
          const cc = f.properties.ISO_A2; const name = f.properties.ADMIN;
          const url = `/projects?country=${cc}`; window.location.href = url;
        })
      });
      layer.addTo(map);
    });
  }
}
```

---

### 8) Landing + Carousel

**src/app/features/landing/landing.component.ts**

```ts
import { Component } from '@angular/core';
import { TranslateModule } from 'ngx-translate-core';
import { MapComponent } from '../map/map.component';
import { RadialWheelComponent } from '../../shared/components/radial-wheel/radial-wheel.component';

@Component({
  standalone: true,
  imports: [TranslateModule, MapComponent, RadialWheelComponent],
  template: `
  <section class="px-6 py-12 max-w-6xl mx-auto">
    <header class="text-center mb-8">
      <h1 class="text-4xl font-bold">{{ 'title' | translate }}</h1>
      <p class="text-lg text-gray-600">{{ 'tagline' | translate }}</p>
      <a routerLink="/projects" class="inline-block mt-4 px-4 py-2 rounded-2xl bg-black text-white">{{ 'browse' | translate }}</a>
    </header>
    <oa-map class="mb-10"></oa-map>
    <oa-radial-wheel (axisSelected)="goAxis($event)"></oa-radial-wheel>
  </section>
  `,
  styles:[":host{display:block}"]
})
export class LandingComponent {
  goAxis(axis:string){ window.location.href = `/projects?axis=${axis}` }
}
```

---

### 9) Data (Static JSON)

**src/assets/data/projects.json** (example)

```json
[
  {
    "slug": "rain-barrel-first-flush",
    "title": "Rain Barrel with First-Flush Diverter",
    "axis": "water",
    "countryCodes": ["KE","GH","SN"],
    "summary": "DIY rain harvesting using food-grade barrel and simple first flush.",
    "budgetMin": 25, "budgetMax": 60, "currency": "USD",
    "difficulty": 2, "durationHours": 4,
    "environment": ["tropical", "temperate"],
    "steps": [
      {"title": "Prep barrel", "body": "Clean thoroughly; mark inlet/outlet."},
      {"title": "Install diverter", "body": "Add vertical pipe with drain valve for first flush."}
    ]
  },
  { "slug": "solar-dryer-cabinet", "title": "Simple Solar Cabinet Dryer", "axis": "food", "countryCodes": ["UG","TZ"], "summary": "Dry fruits/vegetables to improve food security.", "budgetMin": 40, "budgetMax": 120, "currency": "USD", "difficulty": 2, "durationHours": 6, "environment": ["tropical"] }
]
```

---

### 10) Projects List

**src/app/features/projects/list/projects-list.component.ts**

```ts
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';
@Component({
  standalone: true,
  imports: [CommonModule, RouterModule],
  template: `
  <div class="max-w-6xl mx-auto p-6">
    <h2 class="text-2xl font-semibold mb-4">Projects</h2>
    <div class="flex gap-3 mb-4">
      <input class="border rounded px-3 py-2 flex-1" placeholder="Search title" [(ngModel)]="q" />
    </div>
    <div class="grid md:grid-cols-2 gap-4">
      <a *ngFor="let p of filtered()" [routerLink]="['/projects', p.slug]" class="block p-4 border rounded-2xl hover:shadow">
        <h3 class="font-semibold">{{p.title}}</h3>
        <p class="text-sm text-gray-600">{{p.summary}}</p>
        <div class="text-xs mt-2">Axis: {{p.axis}} • Budget: {{p.budgetMin}}–{{p.budgetMax}} {{p.currency}}</div>
      </a>
    </div>
  </div>
  `
})
export class ProjectsListComponent implements OnInit {
  projects:any[]=[]; q='';
  ngOnInit(){
    fetch('/assets/data/projects.json').then(r=>r.json()).then(d=>this.projects=d);
    const url = new URL(window.location.href);
    this.axis = url.searchParams.get('axis')||'';
    this.country = url.searchParams.get('country')||'';
  }
  axis=''; country='';
  filtered(){
    return this.projects.filter(p=>
      (!this.axis || p.axis===this.axis) &&
      (!this.country || (p.countryCodes||[]).includes(this.country)) &&
      (!this.q || p.title.toLowerCase().includes(this.q.toLowerCase()))
    );
  }
}
```

> If `ngModel` complains, import `FormsModule` in `imports`.

---

### 11) Project Detail

**src/app/features/projects/detail/project-detail.component.ts**

```ts
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ActivatedRoute } from '@angular/router';
import { marked } from 'marked';

@Component({
  standalone: true,
  imports: [CommonModule],
  template: `
  <div class="max-w-3xl mx-auto p-6" *ngIf="p">
    <h1 class="text-3xl font-bold mb-1">{{p.title}}</h1>
    <p class="text-gray-600 mb-4">{{p.summary}}</p>
    <div class="space-y-3">
      <div *ngFor="let s of p.steps">
        <h3 class="font-semibold">{{s.title}}</h3>
        <div [innerHTML]="toHtml(s.body)"></div>
      </div>
    </div>
  </div>`
})
export class ProjectDetailComponent implements OnInit {
  p:any; constructor(private route: ActivatedRoute){}
  ngOnInit(){
    fetch('/assets/data/projects.json').then(r=>r.json()).then(list=>{
      const slug = this.route.snapshot.paramMap.get('slug');
      this.p = list.find((x:any)=>x.slug===slug);
    });
  }
  toHtml(md:string){ return marked.parse(md||''); }
}
```

---

### 12) Africa GeoJSON

* Use a lightweight Africa GeoJSON (country polygons) named `assets/geo/africa.geojson` (any public domain source works). Ensure it contains `ADMIN` (name) and `ISO_A2` country code.

---

### 13) Build & Deploy

```bash
npm run build
firebase deploy
```

**firebase.json** (ensure)

```json
{ "hosting": { "public": "dist/one-africa/browser", "ignore": ["firebase.json","**/.*","**/node_modules/**"] } }
```

---

### 14) Smoke Tests

* Landing loads; wheel sectors clickable → `/projects?axis=...`
* Map click → `/projects?country=...` and list filters
* Project detail loads content from JSON
* EN/FR switch works (temporarily add two buttons to test)
* Mobile viewport (≤ 375 px) usable

---

### 15) Next-Day Upgrades (fast wins)

* Replace JSON with Firestore reads (same schema)
* Add star ratings (localStorage or Firestore) and sort by score
* Submission form (draft→pending), moderation flag
* Add more locales (PT, AR, SW) and RTL support
* PWA offline cache + image lazy loading

