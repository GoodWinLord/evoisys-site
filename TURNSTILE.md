
> ВАЖНО (13.11.2026): серверная проверка `ts_token` уже встроена в `leads.py` на VPS
> и ждёт только секрета. Пока секрета нет, форму защищает своя капча-задача
> (proof-of-work) — см. [ANTISPAM.md](ANTISPAM.md).
# Turnstile для формы заявки — серверная часть (api.evoisys.ru)

Фронт уже готов (index.html): виджет рендерится только при реальном ключе,
в JSON заявки добавлено поле `ts_token`. Виджет спит, пока в HTML стоит заглушка `__TS_SITEKEY__`.

## Что нужно
1. CF → Turnstile → Add widget → hostname `evoisys.ru` → получить sitekey + secret
2. Sitekey владелец/облако подставит в index.html (и v2.html) вместо `__TS_SITEKEY__` (2 места: виджет)
3. СЕРВЕР: проверять `ts_token` при каждом POST /api/lead. Без проверки прямой POST в API
   проходит в обход любых фронтовых защит (проверено тестом 28.10).

## Проверка токена (siteverify)
POST https://challenges.cloudflare.com/turnstile/v0/siteverify
Content-Type: application/x-www-form-urlencoded
secret=<TURNSTILE_SECRET>&response=<ts_token>&remoteip=<ip клиента>

Ответ: {"success":true|false, ...}. Принимать заявку только при success:true.
Иначе: 403 {"ok":false,"error":"turnstile"}.

## Python (aiohttp/web)
```python
async def verify_ts(token, ip):
    async with aiohttp.ClientSession() as s:
        async with s.post("https://challenges.cloudflare.com/turnstile/v0/siteverify",
                          data={"secret": TS_SECRET, "response": token, "remoteip": ip}) as r:
            return (await r.json()).get("success") is True
```
В обработчике: `if not await verify_ts(data.get("ts_token",""), request.remote): return 403`

## Node/Express
```js
const r = await fetch("https://challenges.cloudflare.com/turnstile/v0/siteverify", {
  method: "POST", headers: {"Content-Type":"application/x-www-form-urlencoded"},
  body: new URLSearchParams({secret: process.env.TS_SECRET, response: req.body.ts_token, remoteip: req.ip})
});
if (!(await r.json()).success) return res.status(403).json({ok:false});
```

## Тест после установки
1. Открыть сайт, форма должна показать Turnstile-виджет
2. Отправить заявку → приходит в Telegram
3. Прямой POST без ts_token → должен вернуть 403:
   curl -X POST https://api.evoisys.ru/api/lead -H 'Content-Type: application/json' -d '{"name":"тест","phone":"+70000000000","msg":"x","page":"x","honey":"","ts_token":""}'
