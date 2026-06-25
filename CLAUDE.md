# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## How to work with the owner of this project

The owner builds apps but is **not a software developer**. These are her confirmed preferences — follow them exactly in every session:

### שפה / Language
תקשורת בעברית. קוד ושמות טכניים — באנגלית. אל תערבב שפות בתוך משפט אחד.

### לפני שמתחילים פיצ'ר חדש
כשמבקשים לבנות פיצ'ר חדש — שאל **שאלות הבהרה קצרות** לפני שמתחילים לכתוב קוד. ודא שהבנת נכון את הכוונה.

### הסברים טכניים
**מינימום הסברים.** אל תסביר מה הקוד עושה שורה-שורה. אמור רק מה השתנה ואיפה — משפט אחד קצר מספיק.

### כשיש כמה דרכים לפתור בעיה
**בחר בעצמך את הדרך הטובה ביותר** ופשוט עשה אותה. אל תציג אפשרויות ואל תשאל.

### אישורים לפני שינויים
**פשוט תעשה.** אין צורך בבקשת אישור לפני כל שינוי. תבצע, ואז דווח בקצרה מה עשית.

### באגים שמתגלים תוך כדי עבודה
אם גילית באג **בזמן שאתה עובד על משהו אחר** — ציין זאת בקצרה ותקן גם אותו.

### הצעות שיפור יזומות
**הצע שיפורים גם כשלא ביקשו.** אם רואה משהו שאפשר לשפר — ציין את זה בסוף התגובה כהצעה קצרה.

### פיצ'רים גדולים
**הכל בבת אחת.** אל תחלק לשלבים אלא אם הפיצ'ר ארוך במיוחד. תגמור את הכל ואז תדווח.

### עדיפות בכתיבת קוד
**איכות על פני מהירות.** קוד נקי, מסודר ונכון — גם אם לוקח קצת יותר.

### בדיקה אחרי שינויים
אחרי שאתה מסיים, **ציין בקצרה מה צריך לבדוק בדפדפן** (לדוגמה: "פתחי את האפליקציה, הוסיפי ארוחה ובדקי שמופיעה ברשימה"). הבעלים בודקת בעצמה.

---

## What this is

NutriTrack ("Nutri") is a personal nutrition-tracking PWA for a specific patient named **צוף (Tzof)** — a 44-year-old Israeli woman with pre-diabetes, fatty liver, and inflammation tendency. The entire app lives in a **single file: `index.html`** (~3700 lines). There is no build system, no package manager, no framework. Open `index.html` in a browser to run it.

## Development

Since there is no build step, development is: edit `index.html`, refresh the browser.

To test AI features, an Anthropic API key starting with `sk-ant-` must be entered via the in-app settings (⚙️ button). The key is stored only in `localStorage` under `nutri_api_key`.

To simulate localStorage state (meals, history, blood tests), use browser DevTools → Application → Local Storage.

## Architecture

### File layout inside `index.html`
1. `<head>`: Inline PWA manifest generation, CDN load of `marked.min.js` (Markdown renderer)
2. `<style>`: All CSS, including CSS variables (color palette), animations, and RTL layout rules
3. `<body>`: Static HTML structure — topbar, sky scene, day strip, meal/activity lists, bottom nav, all modal shells
4. First `<script>` block: All application logic (~2500 lines of vanilla JS, global scope)
5. Remaining HTML after `</script>`: Additional modal shells (blood tests, history, settings, achievements, etc.)

### UI navigation model
There is no URL routing. The app uses **bottom-sheet modals** toggled by `classList.add('open')` / `classList.remove('open')` on `.modal-overlay` elements. The bottom nav bar's 6 tabs each call a function that populates and opens the relevant modal.

### Data persistence
All state lives in `localStorage`. Keys:

| Key | Contents |
|-----|----------|
| `nutri_meals` | Today's meals array `[{time, desc, userText, productDesc, image, analysis}]` |
| `nutri_activities` | Today's activities `[{type, duration, intensity, time}]` |
| `nutri_all_meals` | Object keyed by `date.toDateString()` → meal arrays (historical) |
| `nutri_all_activities` | Same structure for activities |
| `nutri_history` | Array of `{date, dateKey, type ('interim'|'final'), meals, activities, feedback}` |
| `nutri_blood_tests` | Array of blood test entries, newest first |
| `nutri_trend` | `{date, trend ('up'|'neutral'|'down'), sugar, chol}` for today |
| `nutri_usage` | `{input, output, calls}` token counters |
| `nutri_saved_meals` | Quick-chip meal labels (user-editable list) |
| `nutri_saved_products` | Scanned packaged products with images (max 6, FIFO) |
| `nutri_achievements` | `[{dateKey, date, type ('trophy'|'badge'|'medal'), ...}]` |
| `nutri_last_cleared` | Snapshot of meals+activities from last completed day (for restore) |
| `nutri_api_key` | Anthropic API key |

### AI integration

Two Claude models are called directly from the browser using `anthropic-dangerous-direct-browser-access: true`:

- **`claude-sonnet-4-6`**: Main summaries (interim/final) and multi-turn follow-up conversations. Called via `callNutri()` and `callNutriWithMessages()`, both using streaming SSE.
- **`claude-haiku-4-5-20251001`**: Product label image analysis (200 max tokens, returns JSON).

**System prompt** (`NUTRI_SYSTEM`, line ~992): Hardcoded patient profile, dietary guidelines, language rules (Israeli Hebrew, not archaic), glycemic analysis requirements, and a mandatory structured output format. `buildSystemPrompt()` (line ~3179) dynamically replaces the blood test section with the latest values from localStorage before each call.

**Structured output format** the AI must include at the end of every summary:
```
[TREND:חיובי|ניטרלי|שלילי]
===SUGAR===
<4-sentence sugar/glycemic paragraph>
===CHOL===
<4-sentence cholesterol/fat paragraph>
===END===
```

`handleSummaryResult()` parses this structure: the TREND drives confetti color and trend card arrows; SUGAR/CHOL text populates the tappable info cards on the home screen; everything before the markers is the readable AI summary shown to the user.

### Day lifecycle

1. User logs meals (`saveMeal()`) and activities (`saveActivity()`) throughout the day.
2. "סיכום ביניים" (interim) calls the AI mid-day; result is saved to history as `type:'interim'`.
3. "סיכום יומי סופי" (final summary) calls the AI end-of-day. After response: today's data is archived to `nutri_all_meals`/`nutri_all_activities`, a restore snapshot is saved to `nutri_last_cleared`, and `nutri_meals`/`nutri_activities` are cleared. A trophy is awarded if the trend was positive.

### Achievements system

- 🏆 **Trophy**: awarded by `awardDailyTrophy()` when a final summary returns `[TREND:חיובי]`
- 🏅 **Badge**: awarded by `checkWeeklyBadge()` when 4+ trophies exist in the same calendar week
- 🥇 **Medal**: awarded by `checkMonthlyMedal()` when the majority of weeks in a month have badges

### Sky scene

`buildSkyScene()` renders an animated time-of-day scene (morning 5am–1pm, noon 1pm–4:30pm, evening 4:30pm–8:30pm, night otherwise) using DOM-injected SVG and CSS animations. The scene refreshes every 5 minutes and is swipeable left/right to preview other times of day.

## Key conventions

**Language**: The UI and all AI responses are in Hebrew. The app is RTL (`html dir="rtl"`). When modifying text strings or adding new UI, use Israeli Hebrew consistent with the existing tone (professional-medical, not colloquial).

**Inline styles vs. CSS**: The codebase uses a mix of CSS-class-based styling (defined in `<style>`) and inline styles on dynamically generated HTML strings. CSS variables (`--peach`, `--cream`, `--text`, etc.) are available in both. Prefer using existing CSS classes when possible.

**DOM manipulation pattern**: Dynamic content is built as HTML string concatenation and assigned via `.innerHTML`. Event handlers are written as inline `onclick="..."` attributes in these strings.

**Image handling**: Product images are compressed to max 600px and JPEG 0.7 quality via `compressImage()` before storage. When localStorage quota is exceeded during save, the fallback strips image data and keeps only `{time, desc}`.

**Modal pattern**: All modals are `.modal-overlay` divs. Open: `el.classList.add('open')`. Close: `closeModal(id)`. Overlay click also closes. The `.modal` class inside gets `.animating` for slide-in animation.

**API pricing constants** (line ~3114):
```js
const PRICE_INPUT  = 3.00  / 1_000_000;  // claude-sonnet-4-6 input
const PRICE_OUTPUT = 15.00 / 1_000_000;  // claude-sonnet-4-6 output
```
Update these if switching models or when pricing changes.
