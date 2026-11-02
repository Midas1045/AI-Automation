## Naira Multi-Currency Rate Bot (n8n + Telegram)

An n8n workflow that fetches live exchange rates and sends a clean, professional message to Telegram showing what 1 unit of each of 15 major currencies is worth in Nigerian naira (NGN), with country flags and today's date.

### Workflow screenshot
<img width="1444" height="368" alt="image" src="https://github.com/user-attachments/assets/3e579037-c661-44f9-9e1e-35b7d2432faf" />

### Results screenshot
<img width="774" height="640" alt="image" src="https://github.com/user-attachments/assets/197c374e-e407-41ff-8a17-007bf365d595" />


### How it works
Schedule Trigger -> HTTP Request -> Edit Fields -> Telegram (Send Message)
     1 item           1 item          1 item           1 message

1. Schedule Trigger starts the workflow at the times I set.
2. HTTP Request calls the ExchangeRate-API open endpoint (https://open.er-api.com/v6/latest/USD). It returns one item containing a rates object, where every          currency is quoted against the US dollar.
3. Edit Fields passes the rates object on to the next node (with Include Other Input Fields turned on).
4. Telegram sends the formatted message using Parse Mode HTML. An expression in the Text field converts the rates to naira and builds one line per currency.

#### Example message
```
Daily Exchange Rate Update

Please find today's indicative rates against the Nigerian naira:

🇺🇸 US Dollar (USD): ₦1,345.47 🇪🇺 Euro (EUR): ₦1,502.33 🇬🇧 British Pound (GBP): ₦1,800.96 🇯🇵 Japanese Yen (JPY): ₦8.54 ...and 11 more currencies

Rates are indicative and may vary at the time of transaction.

The figures above are an example from one day's data.

The 15 currencies

USD, EUR, GBP, JPY, CNY, AUD, CAD, CHF, HKD, SGD, SEK, KRW, NOK, NZD and INR. These are among the most traded currencies in the world. The list is easy to change: add or remove one line in the expression.

```
#### The key expression
```
{{ (() => {
  const r = $json.rates;
  const names = {
    USD: '🇺🇸 US Dollar', EUR: '🇪🇺 Euro', GBP: '🇬🇧 British Pound',
    JPY: '🇯🇵 Japanese Yen', CNY: '🇨🇳 Chinese Yuan', AUD: '🇦🇺 Australian Dollar',
    CAD: '🇨🇦 Canadian Dollar', CHF: '🇨🇭 Swiss Franc', HKD: '🇭🇰 Hong Kong Dollar',
    SGD: '🇸🇬 Singapore Dollar', SEK: '🇸🇪 Swedish Krona', KRW: '🇰🇷 South Korean Won',
    NOK: '🇳🇴 Norwegian Krone', NZD: '🇳🇿 New Zealand Dollar', INR: '🇮🇳 Indian Rupee'
  };
  return Object.keys(names).map(c =>
    names[c] + ' (' + c + '): ₦' +
    (r.NGN / r[c]).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })
  ).join('\n');
})() }}

```

Why it works: the API quotes every currency per US dollar, so rates.NGN means "naira per 1 USD". Dividing it by rates.GBP (pounds per 1 USD) gives naira per 1 pound. The names list adds the flag and country name, because the API only returns three-letter codes.

### Tools used
* n8n (workflow automation)
* ExchangeRate-API open endpoint (rates)
* Telegram Bot API (delivery)

### Setup
* Create the workflow in n8n with the four nodes above.
* Create a Telegram bot with @BotFather, add the token as a Telegram credential in n8n, and press Start on the bot in Telegram.
* Put your numeric Telegram chat ID in the Telegram node.
* In the Telegram node, set Parse Mode to HTML and paste the message with the expression above into the Text field (Expression mode).
* Set the Schedule Trigger times and the workflow timezone, then switch the workflow to Active.

### What I learned
* Item counts help debugging. The HTTP node returned one item containing every rate, so the Aggregate node I first added wasn't needed. Aggregate only helps when    you need to combine many items into one.
* Edit Fields drops everything you don't map. With Include Other Input Fields off, the rates object never reached Telegram. Field names are also case-sensitive      (Rates and rates are different).
* Converting a base currency. Rates quoted against USD can be turned into naira rates by dividing, and the same trick works for any pair.
* Formatting numbers and lists with JavaScript inside n8n expressions, and using Telegram's HTML mode for bold text.

### Limitations
* The free open endpoint updates rates once a day and requires crediting ExchangeRate-API, so the figures are indicative, not live trading prices.
* Currencies with a very small unit value, such as the yen and the won, show small naira amounts because they are quoted per single unit.

### Possible improvements
* Read a list of recipients from Google Sheets and send to several people or groups.
* Add rate-change alerts, for example a message when the dollar moves by more than a set amount.
* Log each day's rates to a sheet to build a history and chart.
* Add other channels such as email or WhatsApp.

### Notes
Do not publish your Telegram chat ID, bot token, or any API key. Replace them with placeholders in any exported workflow.

