# AIQuotaBar — észrevételek az újabb (2.1) verzióhoz képest

**Címzett:** Toprak (AIQuotaBar maintainer)
**Dátum:** 2026-09-30  
**Környezet:** macOS, helyi fork (`wjani63/AIQuotaBar`), napi használat Claude + ChatGPT + Cursor + Copilot figyelésre

**Kapcsolódó GitHub issue (upstream):** https://github.com/yagcioglutoprak/AIQuotaBar/issues/32

---

## Rövid összefoglaló

Az **upstream 2.1.0** (webes panel, átdolgozott UI) funkcióban sokat javít, de a **napi használatban** szembemenő pontok vannak: a menüsáv-viselkedés, a ChatGPT csatlakozás forrása, és a macOS jogosultságok kezelése. A **saját „vivid colors” fork** (natív nagy panel, erősebb színes menü) jobban illik a megszokott workflow-hoz, de **hiányzik belőle** több 2.1-es javítás és a helyileg már működő ChatGPT-fallback.

---

## 1. UI és interakció (2.1 vs. megszokott)

| Téma | 2.1 / main | Megjegyzés (vivid / megszokott) |
|------|------------|----------------------------------|
| Panel | WKWebView / web panel, **bal klikk** | **2.1 hiba (macOS fullscreen):** ha egy app **natív teljes képernyőn** fut (zöld gomb → **külön Space**, nem böngésző F11), **bal klikkre a web panel gyakran nem jelenik meg** — ez **nem** a vivid fork. A 2.1 kódban van `fullScreenAuxiliary`, mégsem megbízható (WebKit betöltés, placement, status item ezen a Space-en). Normál (ablakos) módban más. |
| Menü | Jobb klikk / fogaskerék | Vivid: klasszikus erős színes menü + **natív panel**; **normál módban** a bal klikkes panel használható |
| Menüsáv színek | Halványabb brand színek | Vivid: erősebb, jobban olvasható színek |
| Activity Monitor | Korábban „Python” | 2.1-ben javítva — pozitív |
| Asztali widget | Aláírás / frissítés gond | Nálunk kikapcsolva |

**Kérés / javaslat:** választható UI (web vs. classic/vivid natív). **2.1:** **macOS fullscreen Space** + bal klikk **kötelező teszt** (pl. Cursor/Chrome zöld gomb); ha a web panel nincs ready, ne maradjon néma.

---

## 2. ChatGPT — a legfontosabb működési rés

### Mi történt valójában (napló + config alapján)

- A **Copilot** és **Cursor** továbbra is **böngészősütiből** auto-detectálódik (Chrome) — ez nálunk stabil.
- A **ChatGPT böngészős session** gyakran **„stale”** vagy **egyáltalán nem olvasható** (`cookie-detect` üres lista), miközben a felhasználó **be van jelentkezve** (ChatGPT / Codex oldalról).
- Amikor még **látszott** a ChatGPT a panelen, a napló **nem** azt írta, hogy friss böngészős süti működött, hanem több száz alkalommal:  
  **`ChatGPT: using ~/.codex/auth.json token (browser session stale)`**  
  Tehát a **működő megoldás a Codex CLI `~/.codex/auth.json` fájlja** volt (access_token + account_id), nem a NextAuth süti.
- **2.1 / vivid visszaállítás** (~2026-09-30 12:18) után ez a fallback **eltűnt a futó kódból** → `fetch_chatgpt` **401**, üres detect → menüben **„+ ChatGPT”**, pipa nem jön be.
- **Javítás (helyi, vivid branch):** visszarakva a **Codex auth.json olvasás** böngésző után / helyette; config jelölő: `__codex_auth__`; **újra működik** a menüben.

### Upstream / 2.1 oldal

- **#16 chunked NextAuth** (`__Secure-next-auth.session-token.0`) — main-en megvan; a **régebbi fork providers** nélkülözte → nagy sessionnél **teljes kimaradás**.
- **Multi-workspace ChatGPT:** stash / feat ág szerint kell **`ChatGPT-Account-Id`** fejléc a wham API-hoz — érdemes main-be merge-elni.
- **ChatGPT asztali app ≠ böngésző süti:** sok user csak appot használ; **browser-only detect önmagában nem elég**.

**Kérés / javaslat (upstream felé):**

1. **Hivatalos fallback:** ha nincs érvényes böngészős session, olvassa a **`~/.codex/auth.json`**-t (ahol a Codex CLI úgyis tárol tokeneket) — ugyanazzal a log üzenettel, mint korábban.
2. Chunked cookie detect **minden release ágon**.
3. API Providers menü: sikeres Codex-fallback esetén is **pipa** (`chatgpt_cookies` legyen beállítva, akár markerrel).
4. Dokumentáció: „ChatGPT-hez: böngésző **vagy** Codex CLI login”.

---

## 3. macOS jogosultságok (TCC / Full Disk Access)

- LaunchAgent jelenleg: **`~/.ai-quota-bar/.venv/bin/python3`** + `claude_bar.py` — **nem** a `.app` bundle.
- **Full Disk Access** a Menu.app-ra **nem elég**, ha a folyamat továbbra is a venv Python — a **tényleges futó binárist** kell engedélyezni (vagy egységes `.app` launcher, plist-ben).
- Chrome **App-Bound Encryption** miatt időnként „Unable to read database file” — böngésző olvasás **fragilis**, ez erősíti a **Codex fájl fallback** indokát.

**Kérés:** egyértelmű telepítési út: **aláírt app** + plist, vagy dokumentált FDA célpont (melyik executable).

---

## 4. Menü technikai hiba (vivid + panel hook)

- Ha a status item menüje **`setMenu_(None)`** (bal klikk = panel), a **jobb klikkes / popup menü** almenü elemei (pl. **API Providers → ChatGPT**) néha **nem kattinthatók** vagy nem frissül a pipa, mert csak a felső szint kap `setEnabled_(True)`-t.
- **Javítás:** rekurzív `_finalize_popup_menu` az almenükre is; provider detect **háttérszálban**, ne blokkolja a kattintást.

---

## 5. Verzió- és branch-helyzet (fejlesztői)

| Ág | Jelleg |
|----|--------|
| `main` / **2.1.0** | Web panel, settings, history, chunked cookie, org uuid javítások |
| `cursor/menu-bar-vivid-colors` | Natív panel, erős menüszínek, widget tiltás — **providers régebbi** |
| `feat/chatgpt-account-id` (stash) | ChatGPT-Account-Id fejléc — nincs teljes merge vivid-be |
| Helyi javítás | Codex `auth.json` fallback + chunked detect + menü enable |

**Kockázat:** main ↔ vivid **szétcsúszás** — UI-t a fork adja, javítások main-en maradnak.

**Javaslat:** egy **„classic UI”** release channel vagy config flag a 2.1 kódbázison, ne külön fork életben tartás.

---

## 6. Tesztelési checklist (következő kiadáshoz)

- [ ] ChatGPT: csak Codex auth.json, böngésző kijelentkezve → wham/usage OK, menü pipa
- [ ] ChatGPT: csak böngésző, chunked `.0` süti → detect OK
- [ ] ChatGPT: multi-workspace → Account-Id fejléc
- [ ] API Providers almenü kattintható panel-hook mellett
- [ ] Copilot + Cursor továbbra is detect + fetch
- [ ] FDA: venv vs .app — dokumentált működő kombináció
- [ ] **AIQuotaBar 2.1 + macOS fullscreen (külön Space):** bal klikk menüsáv → **web panel** megjelenik

---

## 7. Összegzés Topraknak

A **vivid UI** nálunk **ablakos módban** bevált; az **új 2.1** viszont **macOS natív fullscreen** mellett (külön Space) a **bal klikkes web panel gyakran használhatatlan**. ChatGPT nálunk a **Codex auth fájlra** épült — a 2.1/vivid váltás ezt elvette (helyben visszarakva). Ideális cél: **2.1 funkciók + classic/vivid opció + Codex fallback + fullscreen-barát 2.1 web panel**.

Ha kell, csatolható: `~/.claude_bar.log` időbélyegek (2026-09-29 … codex auth sor; 2026-09-30 12:18 … 401), config `chatgpt_cookies: __codex_auth__` állapot a javítás után.
