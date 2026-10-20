# מוניטור התקפות DDoS גלובלי: לוח בקרה לימודי

דף אינטרנט סטטי אחד (`index.html`) בסגנון חדר בקרה (SOC), שמציג דפוסי התקפות DDoS בין מדינות. מיועד למגמת מערכות תקשוב (כיתות י״א–י״ב). אין שלב בנייה (build): פותחים את הקובץ ועובדים.

```
ddos-monitor/
├── index.html         ← כל הדף: HTML + CSS + JavaScript
├── data/attacks.json  ← קובץ דוגמה למצב snapshot
└── README.md
```

## איך זה עובד: מודל היברידי

| חלק בדף | מקור | אמיתי? |
|---|---|---|
| מפת חום, נתיבים, אזורים חמים, Top 5 | נתונים **מצטברים** מ-Cloudflare Radar: אחוז ההתקפות לכל זוג מקור → יעד | כן (במצבי snapshot/live עם נתונים אמיתיים) |
| יומן האירועים החי | **סימולציה**: אירועים אקראיים, אבל זוגות המדינות נבחרים לפי האחוזים האמיתיים | לא. כל שורה מסומנת `SIM` |

ב-Radar אין התקפות בודדות, חותמות זמן או כתובות IP. לכן כל פרט ביומן (וקטור, IP, עוצמה, משך) מומצא. רק **ההתפלגות** של זוגות המדינות משקפת את הנתונים.

## שלושת מצבי הנתונים

משנים את `CONFIG.DATA_MODE` בראש הסקריפט ב-`index.html`.

### 1. `"demo"` (ברירת מחדל)
נתוני דוגמה מובנים בקוד (25 זוגות לכל שכבה). עובד גם בלחיצה כפולה על הקובץ, בלי שרת ובלי מפתח. התג בראש הדף מציין שאלה **נתוני דוגמה**.
(המפה עצמה נטענת מ-jsDelivr, ולכן צריך חיבור לאינטרנט. בלי חיבור, שאר הלוח ממשיך לעבוד.)

### 2. `"snapshot"`
קורא את `data/attacks.json`. דפדפנים חוסמים `fetch` לקבצים מקומיים (`file://`), ולכן צריך שרת מקומי:

```bash
cd ddos-monitor
python -m http.server 8000
# ואז לפתוח http://localhost:8000
```
(אפשר גם להשתמש ב-Live Server ב-VS Code.)

מבנה הקובץ: `{ "meta": {...}, "layer3": <תשובת Radar>, "layer7": <תשובת Radar> }`.
בקובץ המצורף כתוב `"isSample": true`, כי הערכים בו להמחשה בלבד. הדף מזהה את הדגל ומציג תג "נתוני דוגמה".

**כדי ליצור snapshot אמיתי** (המורה, עם מפתח API משלו; אף פעם לא בתוך הדף):
```bash
TOKEN=...   # Cloudflare API Token עם הרשאת Radar:Read
API=https://api.cloudflare.com/client/v4
curl -s -H "Authorization: Bearer $TOKEN" "$API/radar/attacks/layer3/top/attacks?limit=25&dateRange=1d&format=json" > l3.json
curl -s -H "Authorization: Bearer $TOKEN" "$API/radar/attacks/layer7/top/attacks?limit=25&dateRange=1d&format=json" > l7.json
jq -n --slurpfile l3 l3.json --slurpfile l7 l7.json \
  '{meta:{fetchedAt:(now|todate), source:"Cloudflare Radar (CC BY-NC 4.0)"}, layer3:$l3[0], layer7:$l7[0]}' \
  > data/attacks.json
```

### 3. `"live"`
הדף מושך נתונים כל `DATA_REFRESH_MINUTES` דקות משרת מתווך (Proxy), שכתובתו מוגדרת ב-`CONFIG.PROXY_URL`.

**למה צריך Proxy?** ה-API של Radar דורש מפתח (Token). **כל מה שנמצא ב-JavaScript בצד הלקוח הוא ציבורי**: כל אחד יכול ללחוץ F12 ולהעתיק אותו. לכן המפתח נשמר רק בשרת, כ-Secret.

דוגמה ל-Cloudflare Worker (חינמי):

```js
// worker.js. המפתח נשמר כ-Secret:  npx wrangler secret put RADAR_TOKEN
const ALLOWED_PATHS = [
  "/radar/attacks/layer3/top/attacks",
  "/radar/attacks/layer7/top/attacks",
];

export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    // רשימה לבנה: ה-Proxy מעביר רק את שני הנתיבים האלה, ולא כל בקשה שמגיעה
    if (request.method !== "GET" || !ALLOWED_PATHS.includes(url.pathname)) {
      return new Response("Not found", { status: 404 });
    }
    const upstream = "https://api.cloudflare.com/client/v4" + url.pathname + url.search;
    const res = await fetch(upstream, {
      headers: { Authorization: `Bearer ${env.RADAR_TOKEN}` },
      cf: { cacheTtl: 900, cacheEverything: true }, // מטמון של 15 דקות
    });
    return new Response(await res.text(), {
      status: res.status,
      headers: {
        "Content-Type": "application/json",
        "Access-Control-Allow-Origin": "*", // מומלץ: להחליף בכתובת האתר שלכם
        "Cache-Control": "public, max-age=900",
      },
    });
  },
};
```
אחר כך מגדירים ב-`index.html`:
```js
DATA_MODE: "live",
PROXY_URL: "https://radar-proxy.<your-subdomain>.workers.dev",
```
**בכל שגיאה** (אין Proxy, זמן המתנה, JSON פגום) הדף נופל לנתוני הדוגמה ומציג אזהרה.

## כללי שקיפות (לא לשנות!)

1. **תג תמיד גלוי** בראש המסך: "מפה: נתונים מצטברים של Cloudflare Radar (CC BY-NC 4.0) · יומן אירועים: מדומה, משוקלל לפי נתונים אמיתיים". בנתוני דוגמה התג אומר זאת במפורש.
2. **כתובות IP מדומות רק מטווחי תיעוד** (RFC 5737): `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`. הפונקציה `simulatedSourceIp()` בודקת זאת.
3. **כל שורה ביומן מסומנת `SIM`**, וגם קובץ ה-CSV מכיל עמודה `data_type = SIMULATED`.
4. **"מדינת מקור" אינה התוקף.** ב-L3/4 המקור הוא מיקום מרכז הנתונים של Cloudflare שקלט את התעבורה (כי קל לזייף כתובות IP). ב-L7 המקור הוא לרוב מחשב פרוץ או Proxy. הנתונים משקפים רק את מה שספק אחד רואה.
5. **רישיון:** נתוני Radar הם CC BY-NC 4.0. חובה לציין מקור, ואסור שימוש מסחרי.

## מבנה הקוד

הסקריפט מחולק לסעיפים ממוספרים: `CONFIG` ← `DATA` (טבלת מדינות, נתוני דוגמה, פענוח) ← `MAP` (D3) ← `SIMULATOR` ← `LOG` ← `UI`.
ספריות: `d3@7.9.0`, `topojson-client@3.1.0`, `world-atlas@2.0.2`, כולן בגרסאות נעולות מ-jsDelivr.

## משימות הרחבה לתלמידים

1. **מדינה חדשה בטבלה.** הוסיפו ל-`COUNTRY_TABLE` שלוש מדינות (קוד alpha-2, קוד מספרי ISO, שם ונקודת מרכז), ושנו את נתוני הדוגמה כך שיופיעו במפה. בדקו: מה קורה לנתיב של מדינה שחסרה בטבלה, ולמה?
2. **וקטור תקיפה חדש.** הוסיפו ל-`ATTACK_VECTORS.L3` את `Memcached amplification` (UDP, פורט מקור 11211). חקרו את גודל החבילה הטיפוסי, ובדקו כיצד הוא משפיע על Mpps לעומת Gbps.
3. **גרף קצב.** הוסיפו מעל היומן גרף קווי קטן (D3) של מספר האירועים בכל 10 שניות בדקה האחרונה, צבוע לפי חומרה.
4. **התראות SOC.** כתבו כלל: "אם יש יותר מ-3 אירועים קריטיים לאותה מדינה בתוך דקה, הציגו התראה". דונו: מה ההבדל בין התראה על נתון מדומה להתראה במערכת אמיתית, ומה זה False Positive?
