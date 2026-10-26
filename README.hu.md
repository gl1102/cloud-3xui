[简体中文](README.md) | [English](README.en.md) | [Русский](README.ru.md) | [日本語](README.ja.md) | [한국어](README.ko.md) | [Español](README.es.md) | [العربية](README.ar.md) | [Čeština](README.cs.md) | [Dansk](README.da.md) | [Deutsch](README.de.md) | [Esperanto](README.eo.md) | [فارسی](README.fa.md) | [Suomi](README.fi.md) | [Français](README.fr.md) | [Ελληνικά](README.gr.md) | [Magyar](README.hu.md) | [Bahasa Indonesia](README.id.md) | [Italiano](README.it.md) | [മലയാളം](README.ml.md) | [Nederlands](README.nl.md) | [Norsk](README.no.md) | [Polski](README.pl.md) | [Português](README.ptbr.md) | [Română](README.ro.md) | [Türkçe](README.tr.md) | [Українська](README.ua.md) | [Tiếng Việt](README.vn.md)

# ☁️ cloud-3xui — Ingyenes GitHub VLESS csomópont

A 3x-ui panel + VLESS csomópont automatikus felépítése a GitHub Actions ingyenes szerverein, fix domainnel, automatikus megújítással kapcsolatvesztés esetén.

## Architektúra-diagram

```mermaid
flowchart LR
    subgraph GitHub["GitHub Actions Runner (Ubuntu)"]
        XUI["3x-ui 面板\n:2053"]
        XRAY["Xray VLESS+WS\n:8080"]
    end
    subgraph CF["Cloudflare"]
        TUN["命名隧道\n(固定域名)"]
    end
    USER["用户客户端\nv2rayN / NekoBox 等"]

    XUI --> TUN
    XRAY --> TUN
    TUN --> USER

    style TUN fill:#f6821f,color:#fff
    style USER fill:#1f9d55,color:#fff
```

## Automatikus megújítási folyamat

```mermaid
flowchart TD
    A["手动 Run workflow\n(或上一轮自动触发)"] --> B["安装 3x-ui"]
    B --> C["API 创建 VLESS+WS 入站"]
    C --> D["启动 Cloudflare 命名隧道"]
    D --> E["输出节点链接"]
    E --> F["保活 N 分钟<br/>(手动输入，默认 290)"]
    F --> G["结束前提前触发下一轮<br/>(长运行提前 X 分钟,默认 5<br/>短运行提前 Y 秒,默认 30)"]
    G --> H["新旧 runner 重叠运行<br/>流量无缝切换"]
    H --> A

    style G fill:#f6821f,color:#fff
```

## Automatikus megújítás

- **Az időtartamot te szabod meg**: Run workflow indításakor töltsd ki kézzel a `run_minutes` mezőt, annyi percig fut, amennyit beírsz; üresen hagyva az alapértelmezett 290 érvényes
- **Paraméterek automatikus öröklése**: minden kör továbbadja a `run_minutes` / `countdown_interval` / `countdown_unit` / `overlap_threshold` / `overlap_long` / `overlap_short` értékeket a következőnek, az egész lánc az eredetileg megadott értékeidet használja
- **Átfedő indítás (zökkenőmentes átállás)**: a következő futást korán indítja az aktuális vége előtt, a régi és az új runner egyszerre online van, a Cloudflare tunnel több kapcsolatot támogat, a forgalom automatikusan a működő kapcsolatokra terelődik, a felhasználó szinte észre sem veszi a régi kör kilépését. Az indítás időzítése konfigurálható: a `overlap_threshold` (alapértelmezett 5 perc) értéknél hosszabb keep-alive hosszú futásnak számít, `overlap_long` (alapértelmezett 5 perc) előbb indul; ha rövidebb, rövid futásnak számít, `overlap_short` (alapértelmezett 30 másodperc) előbb indul
- Gyors teszthez írj `5`-öt, napi használatra hagyd üresen (290)
- A domain és az UUID fix, a klienst egyszer kell beállítani
- A leállításhoz nyomj Cancel-t az Actions oldalon· a törlés után nem indul újra automatikusan

## Rögzített címek

| Használat | Cím |
|------|------|
| Csomópont | `vless.mallow.ccwu.cc:443` (VLESS + WebSocket + TLS) |
| Panel | `https://panel.mallow.ccwu.cc/panel` (felhasználónév: `admin`) |

> ⚠️ A fenti domainek csak példák, cseréld le a sajátjaidra.

## Részletes konfigurációs lépések

### Ingyenes domain (nincs domained?)

Nincs saját domained? Ingyen regisztrálhatsz egyet a https://my.dnshe.com/ oldalon (pl. `.ccwu.cc`, 10 év érvényességgel), majd irányítsd a domain NS rekordjait a Cloudflare-re.

### 1. lépés: Elnevezett Cloudflare tunnel létrehozása

Cloudflare hivatalos weboldala: https://www.cloudflare-cn.com (belföldi) / https://www.cloudflare.com (nemzetközi)

Vezérlőpult: https://dash.cloudflare.com

```mermaid
flowchart TD
    A["控制台 → Zero Trust<br/>→ Networks → Tunnels"] --> B["右上角 创建隧道"]
    B --> C["选 Cloudflared<br/>(推荐) → 选择 Cloudflared"]
    C --> D["输入隧道名称<br/>(例：github-3xui)<br/>→ 保存隧道"]
    D --> E["安装并运行连接器页<br/>复制 Token<br/>(eyJ 开头的一长串)"]
    E --> F["下一步 → 路由隧道<br/>→ 已发布应用程序路由"]
    F --> G["添加已发布应用程序路由<br/>子域 vless + 你的域<br/>类型 HTTP → URL 127.0.0.1:8080"]
    F --> H["添加已发布应用程序路由<br/>子域 panel + 你的域<br/>类型 HTTP → URL 127.0.0.1:2053"]
    G --> I["保存完成"]
    H --> I

    style E fill:#f6821f,color:#fff
    style I fill:#1f9d55,color:#fff
```

Részletes lépések:

1. Nyisd meg a [Cloudflare Dashboardot](https://dash.cloudflare.com), majd a bal oldali menüben menj a **Zero Trust** → **Networks** → **Tunnels** menüpontra
2. Kattints jobb felül a **Tunnel létrehozása** gombra
3. **A tunnel típusának kiválasztása**: válaszd a **Cloudflared** lehetőséget (ajánlott), majd kattints a **Cloudflared kiválasztása** gombra
4. **A tunnel elnevezése**: írj be egy nevet (pl. `github-3xui`), majd kattints a **Tunnel mentése** gombra
5. **A csatlakozó telepítése és futtatása**: az oldal megjelenít egy telepítési parancsot — másold ki belőle az `eyJ` karakterekkel kezdődő hosszú **Token** szöveget, és őrizd meg (később GitHub Secret-ként fogod menteni). Helyben **nem kell** telepítened semmit — kattints a Tovább gombra a folytatáshoz
6. **A tunnel irányítása**: lépj a **Közzétett alkalmazásútvonalak** lapra, és kattints a **Közzétett alkalmazásútvonal hozzáadása** gombra — adj hozzá két bejegyzést:
   **Első (csomópont):** Aldomain `vless`; Domain: válaszd ki a saját domained; Típus `HTTP`; URL `127.0.0.1:8080`
   **Második (panel):** Aldomain `panel`; Domain: válaszd ki a saját domained; Típus `HTTP`; URL `127.0.0.1:2053`
7. Mentsd el. A Cloudflare automatikusan létrehozza a DNS-rekordokat, kézzel nem kell hozzáadnod semmit. Térj vissza a Tunnels listához — ha a tunnel állapota **Egészséges (Healthy)**, a beállítás sikeres.

### 2. lépés: UUID generálása

1. Nyisd meg: https://1024tools.com/uuid/
2. Másold ki a generált UUID-t (pl. `cbfc229f-9d34-4fef-9990-8baa82a7327b`)
3. **Mindenképp jegyezd fel ezt az UUID-t**, a workflow naplóiban nem jelenik meg nyílt szövegként, és minden egyes csomópont-link összeállításához szükséged lesz rá

### 3. lépés: GitHub Secrets hozzáadása

1. Lépj be ebbe a repóba, és kattints a **Settings** fülre
2. Bal oldalt **Secrets and variables** → **Actions**
3. Kattints jobb felül a **New repository secret** gombra, és add hozzá sorban a hármat:

   | Name | Secret (érték) | Leírás |
   |------|-------------|------|
   | `CF_TUNNEL_TOKEN` | Az 1. lépésben másolt Tunnel Token | Cloudflare tunnel hitelesítő adat |
   | `PANEL_PASSWORD` | Az általad beállított panel-jelszó | 3x-ui panel bejelentkezési jelszó |
   | `VLESS_UUID` | A 2. lépésben generált UUID | Csomópont UUID, fix, nem változik |

### 4. lépés: A workflow futtatása

1. Kattints a repó tetején az **Actions** fülre
2. Bal oldalt válaszd a **3x-ui VLESS Node** elemet
3. Kattints jobb oldalt a **Run workflow** gombra
4. A felugró űrlapon:
   - **Panel password**: hagyd üresen (automatikusan a `PANEL_PASSWORD` secretet használja)
     Panel-jelszó (üresen hagyva a PANEL_PASSWORD secretet használja)
   - **Run duration**: alapértelmezett `290` (perc), az első teszthez megadhatsz `5`-öt a gyors ellenőrzéshez
     Futási idő (perc, alapértelmezett 290, maximum 340)
   - **Countdown output interval**: visszaszámlálás kimeneti intervallum, alapértelmezett `1`, `0` a kikapcsoláshoz
     Visszaszámlálás kimeneti intervallum (alapértelmezett 1, 0 a kikapcsoláshoz)
   - **Countdown interval unit**: visszaszámlálás intervallum mértékegysége, válasszon `minutes` vagy `seconds` a legördülő menüből
     Visszaszámlálás kimeneti intervallum mértékegysége
   - **Overlap threshold**: átfedési küszöb (perc, alapértelmezett `5`), az ennél hosszabb keep-alive hosszú futásnak számít
     Átfedési küszöb (perc, alapértelmezett 5): az ennél hosszabb keep-alive hosszú futásnak számít
   - **Overlap long**: hosszú futások korai indításának percei (alapértelmezett `5`)
     Hosszú futások korai indításának percei (alapértelmezett 5)
   - **Overlap short**: rövid futások korai indításának másodpercei (alapértelmezett `30`)
     Rövid futások korai indításának másodpercei (alapértelmezett 30)
5. Kattints a zöld **Run workflow** gombra a futtatás indításához

### 5. lépés: A csomópont-link lekérése

1. Várj kb. 2-3 percet, majd kattints a futó feladatra a részletek oldal megnyitásához
2. Bontsd ki a **Start named Cloudflare tunnel** lépést, és nézd meg a naplókimenetet:
   ```
   ==================================================
     面板 Panel: https://panel.mallow.ccwu.cc/panel  (admin / 你的面板密码)

     VLESS 节点 node:
     vless://YOUR_UUID@vless.mallow.ccwu.cc:443?encryption=none&security=tls&sni=vless.mallow.ccwu.cc&type=ws&path=%2Fws#GitHub-3xui

     把 YOUR_UUID 换成你在 VLESS_UUID secret 里填的 UUID
     Replace YOUR_UUID with the UUID from your VLESS_UUID secret
     域名：去 Settings → Secrets and variables → Actions → Variables 改 VLESS_HOST / PANEL_HOST 变量
     Domain: set the VLESS_HOST / PANEL_HOST variables in Settings → Secrets and variables → Actions → Variables
   ==================================================
   ```
3. Másold ki a `vless://` linket, és cseréld le a `YOUR_UUID` részt a saját UUID-dre
4. Illeszd be v2rayN / v2rayNG / Streisand / NekoBox vagy más kliensbe, és már használhatod is

### 6. lépés: Az automatikus megújítás ellenőrzése (opcionális)

1. Az első futtatáskor állítsd az időtartamot `5` percre
2. 5 perc múlva nézd meg az utolsó lépés, a **Trigger next run** naplóját, ennek kell megjelennie:
   ```
   HTTP 204
   已触发下一次运行 Next run triggered
   ```
3. Térj vissza az Actions listára, meg kell jelennie egy új, automatikusan indított futtatásnak
4. Ha minden rendben, futtasd újra, és állítsd vissza az időtartamot `290`-re a normál használathoz

## A workflow lépéseinek leírása

| Lépés | Magyar | English | Leírás |
|------|------|---------|------|
| Install 3x-ui | 3x-ui telepítése | Install 3x-ui | A panel nem interaktív telepítése a hivatalos szkripttel |
| Create VLESS+WS inbound via API | VLESS+WS inbound létrehozása API-n keresztül | Create VLESS+WS inbound via API | Bejelentkezés API Tokennel, az inbound automatikus létrehozása |
| Start named Cloudflare tunnel | Elnevezett Cloudflare tunnel indítása | Start named Cloudflare tunnel | Csatlakozás a fix domainű tunnelhez Tokennel |
| Keep alive | Életben tartás | Keep alive | A megadott percig alszik, percenként ellenőrzi a szolgáltatások állapotát, a beállított időközönként kiírja a visszaszámlálást |
| Trigger next run | A következő futtatás indítása | Trigger next run | API-hívás a következő kör automatikus elindításához |

## Gyakori kérdések

**K: A naplóban a csomópont-link UUID-je `***`-ként jelenik meg?**
V: Ez a GitHub secret-védelmi mechanizmusa. Egyszerűen cseréld le a `YOUR_UUID` részt arra az UUID-re, amit a `VLESS_UUID` secretbe írtál.

**K: A kliens tesztje nem megy át (-1)?**
V: Ellenőrizd három dolgot: ① az UUID-t helyesen cserélted-e le; ② elég új-e a kliens verziója (v2rayN 7.x+); ③ a Cloudflare tunnel rendesen csatlakozik-e (nézd a naplót).

**K: Az automatikus újraindítás nem indult el?**
V: Nézd meg a **Trigger next run** lépés naplóját. Ha nem 204-es HTTP státuszkódot mutat, ellenőrizd a repó Actions jogosultsági beállításait.

**K: Megszakad a kapcsolat a váltáskor?**
V: Átfedő indítást használunk: a következő kör még az aktuális vége előtt elindul, a régi és az új runner egyszerre online, elméletileg észrevehetetlen. Extrém esetben (pl. GitHub sor) lehet rövid ingadozás, a kliens automatikusan újracsatlakozik.

**K: Le akarom állítani az automatikus megújítást?**
V: Menj az Actions oldalra, és állítsd le (Cancel) az aktuális futtatást, többé nem indul el automatikusan.

## Megjegyzések

- **Sebesség**: a forgalom a Cloudflare → GitHub adatközpontokon (USA/EU) keresztül megy, csak ideiglenes/enyhe használatra alkalmas
- **Zökkenőmentes átállás**: átfedő indítással minden kör 5 perccel korábban indítja a következőt, a régi és az új runner egyszerre online, elméletileg a felhasználó semmit nem vesz észre; extrém esetben (pl. GitHub sor) lehet rövid ingadozás
- **Szabályosság**: GitHub Actionsön proxyt futtatni sértheti a ToS-t, fióktiltás kockázatával — mérlegeld magad
- **Biztonság**: ne tedd közzé nyilvánosan a `CF_TUNNEL_TOKEN` / `PANEL_PASSWORD` / `VLESS_UUID` értékeket
