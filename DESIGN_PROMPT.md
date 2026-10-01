# Prompt: MijnKlus-stijl toepassen op Entreetje

Plak dit als prompt in de Entreetje-code-sessie. Vul eerst de huiskleuren in (sectie 2).

---

Restyle de hele Entreetje-app in de "MijnKlus design template". Verzin geen eigen stijl: volg onderstaande regels exact.

## 1. Karakter
Vlak, zakelijk, SaaS-productgevoel. Wit en zacht grijs als ondergrond, de donkere huiskleur als hoofdkleur, de accentkleur **alleen** voor de primaire actie. Geen verlopen, geen gloed, geen decoratieve vlekken, geen glass/blur. Presentatiegevoel komt uit (a) een kleine, scherpe typografische hiërarchie, (b) gekantelde 3D-mockups van het echte product en (c) rustige beweging. Elk beeld/mockup toont informatie uit de pagina zelf, nooit generieke plaatjes.

## 2. Tokens
Scope alles onder een wrapperklasse (bijv. `.en-landing`, vervang `mk-` door `en-`). Gebruik geen Tailwind-kleurklassen voor deze secties.

| token | MijnKlus | Entreetje (invullen) | gebruik |
|---|---|---|---|
| ink | `#0b1f33` | [ ] | koppen, hoofdtekst |
| navy (primair) | `#0f2d5c` | [ ] | hoofdkleur, donkere vlakken, eigen chatberichten |
| navy-2 | `#0a2257` | [ ] | donkerder variant |
| blue / blue-2 | `#1d4ed8` / `#2563eb` | [ ] | kicker, links |
| blue-50 / blue-100 | `#eef3fc` / `#dbe6f8` | [ ] | zachte vlakken, halo's, tegels (lichte tint van primair) |
| slate / slate-2 | `#52607a` / `#8793a6` | [ ] | bodytekst / labels |
| line / line-2 | `#e2e7ef` / `#edf0f5` | [ ] | randen / scheidingslijnen |
| soft | `#f5f7fa` | [ ] | afwisselende sectieband |
| accent / accent-hover | `#f97316` / `#ea580c` | [ ] | ALLEEN primaire CTA + kleine accenten |
| green | `#16a34a` | houden | vinkjes, bevestigd |
| ease | `cubic-bezier(.2,.7,.2,1)` | houden | alle beweging |
| shadow-1 | `0 1px 2px rgba(15,45,92,.06), 0 6px 18px rgba(15,45,92,.06)` | tint op primair | kaart |
| shadow-2 | `0 2px 6px rgba(15,45,92,.06), 0 18px 44px rgba(15,45,92,.12)` | tint op primair | zwevend |

Neutrale tinten (line, soft, slate) mogen blijven; de blauwe tinten (blue-50/100, schaduwen) afleiden uit de Entreetje-primaire kleur.

Hoeken: **8px** knoppen/velden, **10–12px** kaarten (nooit 16px+ op panelen, nooit volledig rond behalve avatars/chips). Rand `1px solid var(--line)`. Schaduw in rust `0 1px 2px rgba(primair,.04)`, hover `0 12px 28px rgba(primair,.10)`.

## 3. Typografie
Inter, 14px basis, `-webkit-font-smoothing: antialiased`.
- Hero-titel: `clamp(1.8rem, 1.2rem + 1.8vw, 2.45rem)`, 700, line-height 1.1, letter-spacing -0.035em, regel 2 in navy.
- Sectietitel (h2): `clamp(1.35rem, 1.05rem + .9vw, 1.8rem)`, 650, -0.025em, `text-wrap: balance`.
- Kicker: 11px, 600, 0.12em, hoofdletters, blauw.
- Lead: 14.5px, line-height 1.6, slate, max 540px.
- Body 13–14px. Maximaal één H1 per pagina.
- Container: max 1120px, padding 0 24px. Sectie-padding 64px 0; hero 40px 0 96px.

## 4. Componenten
- **Knoppen**: hoogte 44px, padding 0 18px, radius 8px, 14px/600, pijl-icoon 16px dat 2px naar rechts schuift bij hover. Varianten: `primary` (accent, wit), `line` (wit, rand, hover blue-50), `navy`, `onnavy` (transparant, witte rand). Focus: 2px outline `#93c5fd`, offset 3px.
- **Pill** (hero-badge): inline-flex, radius 6px, bg blue-50, 12px/600 navy, met groen puntje.
- **Hero**: wit, onderrand line-2; 2 kolommen: links pill + titel + lead + CTA's + vinkjes-rij ("proof", 12.5px, groene vinkjes); rechts een visual.
- **Visual**: gekanteld browservenster (`MkWindow`) of telefoon (`MkPhone`) met inhoud van die pagina. Halo = effen cirkel blue-50 (geen gloed), vloer-schaduw = geblurde ellips. Zwevende chipkaartjes: wit, rand, radius 10px, shadow-2.
- **Secties**: afwisselend wit / soft; optioneel blue-50 of navy (witte tekst, kicker `#9fc0ff`).
- **Kaarten (`panel`)**: wit, rand, radius 10–12px, hover-lift + reveal.
- **Eind-CTA**: altijd één component (`FinalCta`): wit blok, rand blue-100, radius 14px, padding 28px 32px, links titel + vinkjes + beschrijving, rechts kolom met primaire knop + onderschrift. Nooit een eigen navy CTA-blok.
- **FAQ**: `<details>`, antwoorden in de DOM.
- **App-schermen**: chat met navy header, eigen berichten navy, andermans wit; kopblok + sticky zijbalk met actie op detailpagina's; admin-panelen `rounded-[12px] border white`, met echte cijfers.

## 5. Beweging
- Reveal: `.rv` (opacity 0 → 1, translateY 14px → 0, .6s ease), eenmalig via IntersectionObserver, `--d` voor stagger.
- 3D-mockup: `rotateY(-12deg) rotateX(4deg) rotateZ(1deg)`, zweeft `Float` (6s, translate 0 → -8px), intro-animatie bij laden, hover vlakt af.
- Lussen (rijen, typen, balken) 6–10s ease-in-out.
- **Altijd** een `@media (prefers-reduced-motion: reduce)` blok dat animaties/reveal uitzet.
- Mobiel: mockups zichtbaar maar zonder scheve 3D op tekstdichte onderdelen; tikvlakken minimaal 44px; knoppen in hero 100% breed; controleer op 375px.

## 6. Inhoud
Alleen echte product-feiten van Entreetje. Geen verzonnen aantallen, prijzen, reviews of garanties.

## 7. Werkwijze
1. Maak één `landing.css` met tokens + alle klassen hierboven, gescoped onder de wrapper.
2. Bouw herbruikbare componenten: `PageHero`, `Window`/`Phone` mockups, `Section`, `Panel`, `Bullet`, `FinalCta`, `useReveal()`.
3. Pas elke pagina toe op deze componenten; geef elke pagina een eigen mockup met echte pagina-informatie.
4. Checklist per pagina: scope-klasse, hero met eigen mockup, alleen tokens, hoeken 8–12px, accent alleen voor primaire actie, FinalCta als afsluiter, reduced-motion, 375px gecontroleerd.
5. Draai typecheck/tests en toon het resultaat.

Valkuilen (uit MijnKlus): globale light-mode regels kunnen tekstkleuren overschrijven, scope daarom je kleuren met `!important` binnen de wrapper; wit-op-donker expliciet zetten.
