# Banko įmokų peržiūra — sesijos kontekstas

## Užduotis

Perkelti `bankas.html` failą į `tomasja/banko-imoku-perziura` repozitoriją ir toliau jį vystyti.

**Šaltinis:** repozitorija `tomasja/atostogu_skaiciuokle`, šaka `claude/nifty-knuth-enrU7`, failas `bankas.html`

## Kas tai

Viengubo failo (`bankas.html`) įrankis — banko kortele surinktų mokėjimų peržiūra. Veikia lokaliai iš `file://` protokolo, nereikia serverio ar interneto. Visos bibliotekos (JSZip 3.10.1) įkeltos inline.

## Palaikomi bankų formatai

| Bankas | Failų tipas | Atpažinimas |
|--------|------------|-------------|
| **SEB** | `.zip` (ZIP su XML viduje) | `<IFX>` šakninis elementas |
| **Swedbank TDM** | `.xml` | `<posreport>` šakninis elementas |
| **Ashburn** | `.xlsx` | ZIP su `xl/worksheets/sheet1.xml`, stulpelis A = "EKS numeris" |

### SEB IFX XML laukų žemėlapis
- `DepAcctStmtRec > MerchantData` → terminalas, prekybos vieta, miestas
- `DepAcctTrnRec > BankAcctTrnRec` → suma, valiuta, datos, kortelė, kortelės tipas
- `XferId` → autorizacijos kodas, `XferComment` → RRN

### Swedbank posreport XML laukų žemėlapis
- `outlet` atributai: `id`, `name`, `city`
- `terminal` atributas: `id`
- `transaction` elementai: `local_date` (YYYYMMDD), `local_time` (HHMMSS), `hidden_pan`, `ret_ref_nr` (RRN), `auth_code`, `orig_amount`, `fee_amount`, `orig_currency`
- `card_group` atributas `brand` → kortelės tipas

### Ashburn XLSX laukų žemėlapis
- A: EKS numeris, B: adresas, C: miestas, D: aptarnavimo vietos pavadinimas
- E: bankas priėmėjas, F: Terminal ID, G: suma, H: grynieji (0=POS)
- I: valiuta, J: data+laikas (Excel serial number), K: kortelės fragmentas
- L: RRN, M: autorizacijos kodas, N: anuliavimo indikatorius (jei užpildytas — eilutė praleidžiama)
- Data konvertuojama: `(serial - 25569) * 86400000` → JS Date UTC

## Katalogų struktūra kliento serveryje

```
\\dezute\...\Atsiskaitymų kortele ataskaitos\
├── Ashburn\           ← .xlsx failai (Transactions_LP_YYYYMMDD.xlsx)
├── Klix\              ← dar neintegruota
├── SEB kortelės, kurjerinės\  ← .zip failai, struktūra: metai/mėnuo/failai
├── Swedbank TDM\      ← .xml failai (plieniniai, be ZIP)
└── Unisend\           ← dar neintegruota
```

Vartotojas pasirenka pagrindinį `Atsiskaitymų kortele ataskaitos` katalogą — visi formatai aptinkami automatiškai.

## Pagrindinės funkcijos

- **Katalogo nuskaitymas** — `<input webkitdirectory>`, failo tipas nustatomas pagal plėtinį
- **Laikotarpio rinkiklis** — metai/mėnesiai iš `webkitRelativePath`; jei struktūroje nėra metų aplanko, data ištraukiama iš failo pavadinimo regex'u
- **Formato atpažinimas** — `.zip` → SEB, `.xlsx` → Ashburn, `.xml` → tikrinamas šakninis elementas
- **Dublikatų aptikimas** — raktas: `kortelė|suma|data|laikas`
- **Kortelių formatavimas** — visi rodomi `*XXXX` (paskutiniai 4 skaitmenys), funkcija `fmtCard()`
- **`localStorage`** — įsimena paskutinį katalogą ir pasirinktus laikotarpius, rodo "greito įkėlimo" juostą
- **Du rodiniai (skirtukai):**
  - *Operacijos* — detalė lentelė su puslapiavimu, rūšiavimu, 14 stulpelių
  - *Dienos apyvarta* — pivot lentelė (eilutės = vietos, stulpeliai = datos)
- **Eksportas:** CSV ir PDF (abu rodiniai), PDF generuojamas `window.print()` naujame lange
- **Filtrai:** paieška, metai, mėnuo, suma nuo/iki, data nuo/iki, terminalas, kortelės tipas, bankas, tik dublikatai

## Žinomi apribojimai

- Apple Pay / Google Pay neidentifikuojami — tokenizuotos kortelės (DPAN) nesutampa su kliento banko išrašo kortelės numeriu (FPAN); identifikuoti reikia pagal RRN arba autorizacijos kodą
- **Klix** ir **Unisend** formatai dar neintegruoti
