### Предистория: 
Решал я таски на [rootme](https://www.root-me.org/) и попалась таска [XMPP - authentication](https://www.root-me.org/en/Challenges/Network/XMPP-authentication-197). Основная цель таски состояла в том, чтобы по захвату пакетов вытащить пароль, который использовался при авторизации и я пошел искать документацию к тому, как работает аунтефикация клиента. Наткнулся на [xmpp wiki](https://wiki.xmpp.org/) и там же наешь документацию на аунтефикацию клиента [SASL_Authentication_and_SCRAM](https://wiki.xmpp.org/web/SASL_Authentication_and_SCRAM(). Из нее я выяснил то, как происходит распознования того, что клиент знает правильный пароль и что просто так из пакетов пароль я не вытащу. И тут я заинтересовался тем, как вообще происходит подтверждение правильного пароля, собственно это меня и побудило на написание статьи.

### Повествование о том, что такое этот ваш XMPP
XMPP(ранее jabber) - протокол обмена сообщениями с открытым исходным кодом, публикуемый под Apache License 2.0, основан на XML. Позволяет мгновено передавать сообщения и информацию о присутствии, также с помощью расширений позволяет передавать файлы(XEP-0363 HTTP File Upload; XEP-0096 SI File Transfer; XEP-0065 SOCKS5 Bytestreams; XEP-0166: Jingle и т.д.), создавать комнаты(XEP-0045: Multi-User Chat), осуществлять голосовые и видеозвонки(XEP-0166: Jingle), хранить произвольные данные на сервере(XEP-0049 Private XML Storage) и т.д.

### А теперь о том, что я хотел рассказать в этой статье
У XMPP есть 2 способа аунтефикации клиента SASL[^1] и OAuth 2.0. Начнем по порядку, с разбора механизмов аунтефикации SASL, используемые в XMPP. 

[^1]: **SASL** (Simple Authentication and Security Layer) — унифицированный фреймворк для реализации механизмов аутентификации и опционального шифрования при передаче данных по сети.

Механизмы SASL, которые используются в XMPP:
1. External
2. SCRAM (Salted Challenge Response Authentication Mechanism)
3. PLAIN
4. DIGEST-MD5
(раположены в порядке убывания безопасности)

А теперь поподробней про каждый механизм:
External - Этот механизм позволяет клиенту аунтефицироваться с помощью внешних средств. Он позволяет входить в систему без пароля, подтверждая свою личность с помощью уникального цифрового сертификата

SCRAM(На нем я и заострю свое внимание) - Наиболее распространенный и обязательнывй для современного ПО XMPP механизм. Обеспечивает надежную защиту пароля. Что меня зацепило именно в этом механизме, это то, что клиент не передает пароль в открытом виде, а подтверждает свое знание пароля.

Как оно работает?
1. Клиент отправляет имя пользователя, под которым он хочет авторизоваться
2. Сервер отправляет обратно соль[^2] для этого пользователя и количество итераций[^3]
3. Клиент хэширует пароль с заданной солью для заданного количества итераций
4. Клиент отправляет результат обратно
5. Сервер хэширует полученное значение и отправляет клиенту, дабы клиент мог удостовериться в том, что у сервера был пароль/хэш пароля

[^2]: Соль — случайное значение, добавляемое к исходному паролю перед хешированием. Обеспечивает уникальность хеша для одинаковых паролей и защищает от атак по предвычисленным таблицам.
[^3]: Итерации — число повторных применений хеш‑функции к результату предыдущего вычисления. Увеличивает вычислительную сложность подбора пароля, замедляя атаки грубой силы.

Какие вариации SCRAM использует XMPP?
- SCRAM-SHA-256-PLUS
- SCRAM-SHA-1-PLUS
- SCRAM-SHA-256
- SCRAM-SHA-1
(расположены в порядке убывания безопасности, также есть SCRAM-SHA512(-PLUS) и SCRAM-SHA3-512(-PLUS), но на оффициальной вики и документе, на который она ссылается, нету информации о том, насколько они безопасны)

А теперь о том, как оно работает под капотом
1. Сначала нормализуется пароль(испольбзуя [SASLprep](https://datatracker.ietf.org/doc/html/rfc4013))
2. Берется случайная строка(для примера будет 32 байта в Hex кодировке). Это будет clientNonce
3. Клиент отправляет первоночальное сообщение(initialMessage)
   `"n=" .. username .. ",r=" .. clientNonce`
4. Клиент добавляет GS2 заголовок[^4] к initialMessage и кодирует полученное в base64 и отправляет это в качестве своего первого сообщения   
```
<auth xmlns="urn:ietf:params:xml:ns:xmpp-sasl" mechanism="SCRAM-SHA-1">
    biwsbj1yb21lbyxyPTZkNDQyYjVkOWU1MWE3NDBmMzY5ZTNkY2VjZjMxNzhl
</auth>
```
5. Сервер отвечает вызовом. Данные закодированы в base64
```
<challenge xmlns="urn:ietf:params:xml:ns:xmpp-sasl">
    cj02ZDQ0MmI1ZDllNTFhNzQwZjM2OWUzZGNlY2YzMTc4ZWMxMmIzOTg1YmJkNGE4ZTZmODE0YjQyMmFiNzY2NTczLHM9UVNYQ1IrUTZzZWs4YmY5MixpPTQwOTY=
</challenge>
```
6. Клиент декодирует это`r=6d442b5d9e51a740f369e3dcecf3178ec12b3985bbd4a8e6f814b422ab766573,s=QSXCR+Q6sek8bf92,i=4096`
7. Клиент парсит отсюда:
   -  r = Это serverNonce, клиент должен убедиться в том, что он начинается с clientNonce, который он отправил в своем первом сообщении
   - s = Это соль, закодированная в base64
   - i = Это количество итераций
6. Клиент вычисляет:
```
clientFinalMessageBare = "c=biws,r=" .. serverNonce
saltedPassword = PBKDF2-SHA-1(normalizedPassword, salt, i)
clientKey = HMAC-SHA-1(saltedPassword, "Client Key")
storedKey = SHA-1(clientKey)
authMessage = initialMessage .. "," .. serverFirstMessage .. "," .. clientFinalMessageBare
clientSignature = HMAC-SHA-1(storedKey, authMessage)
clientProof = clientKey XOR clientSignature
serverKey = HMAC-SHA-1(saltedPassword, "Server Key")
serverSignature = HMAC-SHA-1(serverKey, authMessage)
clientFinalMessage = clientFinalMessageBare .. ",p=" .. base64(clientProof)
```
9. Клиент кодиурет в base64 `clientFinalMessage` и отправляет это как ответ
```
<response xmlns="urn:ietf:params:xml:ns:xmpp-sasl">

Yz1iaXdzLHI9NmQ0NDJiNWQ5ZTUxYTc0MGYzNjllM2RjZWNmMzE3OGVjMTJiMzk4NWJiZDRhOGU2ZjgxNGI0MjJhYjc2NjU3MyxwPXlxbTcyWWxmc2hFTmpQUjFYeGFucG5IUVA4bz0=
</response>
```
10. Если все прошло успешно, то вы поулчите от сервера ответ `<success>`:
```
<success xmlns='urn:ietf:params:xml:ns:xmpp-sasl'>
	dj1wTk5ERlZFUXh1WHhDb1NFaVc4R0VaKzFSU289
</success>
```
если декодировать из base64 это сообщение, то получим:
`v=pNNDFVEQxuXxCoSEiW8GEZ+1RSo=`
11.  Клиент должен убедиться, что значение v - это serverSignature закоидрованный в base64.
И вот и все, дальше можно добавить привязку канала(channel binding), для улучшения защиты(предотвращения проведения MitM атак). 

Plain - 

[^4]: **GS2‑заголовок** — короткий префикс, добавляемый клиентом к своему первому сообщению в процессе аутентификации. Является частью фреймворка GS2, обеспечивающего совместимость между механизмами SASL (Simple Authentication and Security Layer) и GSS‑API (Generic Security Service Application Program Interface). Может содержать флаги `gs2-cb-flag` и `gs2-authzid`.