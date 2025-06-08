## task: finding_my_way
```
Answer the Way number in OpenStreetMap of the building located at 34.735639, 138.994950. Flag Format: Diver25{123456789}
```
Solve: Идем на OpenStreetMap вводим данные нам коорды, в слоях выбираем данные карты, чтобы отображались way id
![[Pasted image 20250607130143.png]]
наше здание
Flag: Diver25{568613762}

## task: hidden_service
```
添付ファイルを確認して、Flagを獲得してください！  
See the attached file and capture the flag!  
Flag形式 / Flag Format: `Diver25{xxxxxxxxxxxxxxxxx}`

Проверьте вложения и выиграйте флаг!  
Посмотрите прикрепленный файл и захватите флаг!  
Формат флага: Diver25{xxxxxxxxxxxxxxxxx}
```
вот фотография из условия ![[hidden_service.jpg]]Solve: Нам дают onion линк, переходим в tor браузер(в моем случае whonix), вбиваем ссылку и нас перекидывает на сайт
![[Pasted image 20250607130454.png]]

Flag: Diver25{w3lc0m3_70_d4rkw3b!}

## task: flight_from
```
Answer with the ICAO code (4 letter code) of the airfield from which this helicopter departed.  
Flag Format: `Diver25{RJTT}`
Image:  
https://drive.google.com/file/d/14LSICiwmUqG9xqAfAm75IVdGd7P12uoy/view?usp=sharing
```
Фотография из задания ![[flight_from.png]]
Solve: Можно сделать логическое предположение, что вертолет взлетал сверху и садился внизу, потому что снизу он сделал несколько кругов(ждал разрешения на посадку)
![[Pasted image 20250607132620.png]]
смотрим на опорные пункты и видим город Tachikawa, заходим на gmaps приближаем туда и там правда есть аэропорт(вот это да:D), захотим на [world-airport-codes](https://www.world-airport-codes.com/) дабы увидеть ICAO код этого аэропорта, ввожу Tachikawa и он радостно предоставляет код
![[Pasted image 20250607133019.png]]

Flag: Diver25{RJTC}
## task: ship
```
This is a vessel operated by a some organisation. Answer the number that would remain the same if this vessel were to be sold to a foreign country in the future.  
Flag Format: `Diver25{ship name in **the local language**_number}` (e.g. If the ship name is "ペンギン饅頭号" and the number is `1234567`, the flag should be `Diver25{ペンギン饅頭号_1234567}`.)
```
прикрепленная фотка ![[ship.jpg]]
Solve: Видим позывной корабля 7JVV, вбиваем в [трекер кораблей](https://www.marinetraffic.com/) и видим такую картину 
![[Pasted image 20250607134324.png]]
Немного гугла не помешают и выясняем, что код IMO не меняется за время жизни судна, ни при смене владельца, ни команды, ни флага, под которым корабль плавает.
закидываем название корабля в переводчик и получаем `神鷹丸`


Flag: Diver25{神鷹丸_9767675}
## task: document
```
The US Navy Commander Fleet Activities Yokosuka (CFAY) operates a shuttle bus service between Haneda airport/Narita airport and the base for US military personnel. Answer the name of the person who created the document about the boarding location of the bus in 2023.  
Flag Format: `Diver25{George Washington}`
```
Solve:

## task: object
```
There is a large object at 69.216246, 33.378242. Answer the project number and particular name of that object in **local language**. Flag Format: `Diver25{PROJECT NUMBER_NAME}` (e.g. `Diver25{955А_Борей-А}`)
```
Solve: координаты ведут нас на побережье базы вмс Оленья Губа
![[Pasted image 20250607151250.png]]
ищем фотографии с оленьей губы и какие проекты там располагаются, поиски наводят на сайт http://militaryrussia.ru/blog/topic-822.html (вообще, изначально наткнулся на ПД-78), все, сдаем таску

Flag: Diver25{13560_ПД-72}

## task: louvre
```
Answer the vendor of one of the Louvre's public Wi-Fi access points that meets the following criteria.

- Information was collected on 28 February 2025. This can be accessed via online.
- Determine the vendor according to the BSSID.

Flag format: `Diver25{Company Name}` (e.g. If the company is Apple Inc., the flag should be `Diver25{Apple Inc.}`.)
```

Solve: сразу же идем в гугл ищем, что такое louvre, понимаем что это музей. Дальше читаем условие, видим собраны какие-то данные, данные связанные с Wi-Fi, делаем вывод, что нужно найти точку доступа этого музея, идем на [WiGLE.net](https://wigle.net/) вписываем в поиск louvre, находим наш музей и ищим точки доступа как-то связаннаые с музеем.
![[Pasted image 20250607230221.png]] 
Дальше идем на любой определитель вендора по MAC адресу у меня это [macaddress.io](https://macaddress.io/) переписываем MAC адрес и получаем вендора `Hewlett Packard Enterprise`

Flag: Diver25{Hewlett Packard Enterprise}

