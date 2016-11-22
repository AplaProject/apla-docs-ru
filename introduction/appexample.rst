################################################################################
Пример припложения
################################################################################

Рассмотрим пример приложения, которое добавляет поле аватар в таблицу citizens у государства, чтобы граждане могли указывать изображения для своего аккаунта. Ниже приведен полный исходный код приложения и затем мы рассмотрим назначение каждой фукнции.

.. code:: js

GetRow(sc, #state_id#_smart_contracts, name, TXEditProfile )
SetVar(
 global = 0,
 typeid = TxId(EditContract),
 typecolid = TxId(NewColumn),
 sc_value = `contract TXEditProfile {
 tx {
  FirstName  string
  Image string "image"
 }
 func main {
   DBUpdate(Table( "citizens"), $citizen, "name,avatar", $FirstName, $Image)
     Println("TXEditProfile new")
 }
}`
)
TextHidden( sc_value, sc_conditions )
Json(`Head: "Adding avatar column",
 Desc: "This application adds avatar column into citizens table.",
 OnSuccess: {
  script: 'template',
  page: 'government',
  parameters: {}
 },
 TX: [
  {
  Forsign: 'global,id,value,conditions',
  Data: {
   typeid: #typeid#,
   type: "EditContract",
   global: #global#,
   id: #sc_id#,
   value: $("#sc_value").val(),
   conditions: $("#sc_conditions").val()
   }
    },
    {
  Forsign: 'table_name,column_name,permissions,index',
  Data: {
   type: "NewColumn",
   typeid: #typecolid#,
   table_name : "#state_id#_citizens",
   column_name: "avatar",
   permissions: "$citizen == #wallet_id#",
   index: 0
  }
  }
 ]
`)

.. code:: js
GetRow(sc, #state_id#_smart_contracts, name, TXEditProfile )
Здесь мы получаем значение полей существующего контракта TXEditProfile из таблицы smart_contracts государства. В этой таблице хранятся все конракты государства. #state_id# является предопределенной переменной и равна индексу государства в которое зашел гражданин. Поиск ведется по полю **name** и значению *TxEditProfile*. Первый параметр sc определяет префикс, который будет добавлен слева к именам полученных колонок. Например, значение поля **value** будет записано в переменную #sc_value#.

.. code:: js
SetVar(
 global = 0,
 typeid = TxId(EditContract),
 typecolid = TxId(NewColumn),
 sc_value = `contract TXEditProfile {
...
}`
)

Функция **SetVar** служит для присваивания значений переменным. **global** - переменная необходимая для транзакции, переменным typeid и typecolid присваиваются идентификаторы транзакций *EditContract* и *NewColumn*. Это те две транзакции, которые будут использоваться в нашем приложении. Первая транзакции изменяет сам контракт TXEditProfile (текст нового контракта мы сохраняем в переменной **sc_value**) на новую версию с учетом загрузки изображений для аватара. С помощью второй транзакции *NewColumn* мы добавим колонку avatar в таблицу citizens у текущего государства. Следует заметить, что получив текущие значения в функции GetRow, переменная sc_value у нас уже содержала текущий текст контракта. В данном случае, мы изменили ее значение на новый вариант. 

.. code:: js
TextHidden( sc_value, sc_conditions )

Функция TextHidden у нас создает скрытые тэги input с указанными значениями и иентификаторами. В данном случае, будет создано два скрытых поля: <input type="hidden" id="#sc_value" value="#sc_value#"> и <input type="hidden" id="#sc_conditions" value="#sc_conditions#">. **sc_conditions** содержит текущее значение поля condition, которое мы получили ранее в функции GetRow.

.. code:: js
Json(`Head: "Adding avatar column",
 Desc: "This application adds avatar column into citizens table.",
 OnSuccess: {
  script: 'template',
  page: 'government',
  parameters: {}
 },
 ...
 
Сама страница приложения формируется с помощью функции Json, которая просто переводит данные в объект на JavaScript. В начале мы указываем заголовок **Head** и описание **Desc** для нашего приложения. Параметр **OnSuccess** определяет на какую страницу следует переходить в случае успешного завершения приложения. Тут определен переход на template страницу с именем government. 

.. code:: js
TX: [
  {
  Forsign: 'global,id,value,conditions',
  Data: {
   typeid: #typeid#,
   type: "EditContract",
   global: #global#,
   id: #sc_id#,
   value: $("#sc_value").val(),
   conditions: $("#sc_conditions").val()
   }
    },
   ...
   
Массив TX определяет последовательность выполняемых транзакций в приложении. Рассмотрим транзакцию *EditContract*. В поле Forsign мы перечисляем параметры транзакции, которые будут подписываться и затем проверятся при обработке блока. Параметр *Data* определяет данные транзакции. Они свои для каждой транзакции за исключением **type** и **typeid** - это наименование и идентифкатор транзакции. В поля **value** и **conditions** мы будем подставлять значения из определенных ранне скрытыфх полей sc_value и sc_conditions.

.. code:: js
{
  Forsign: 'table_name,column_name,permissions,index',
  Data: {
   type: "NewColumn",
   typeid: #typecolid#,
   table_name : "#state_id#_citizens",
   column_name: "avatar",
   permissions: "$citizen == #wallet_id#",
   index: 0
  }

После того, как мы определили транзакцию *EditContract*, мы описываем транзакцию *NewColumn*. И снова кроме стандартных полей **type** и **typeid** мы указываем поля необходимые для данной транзакции. *#wallet_id#* - это еще одно значение по умолчанию, которое равно номеру аккаунта текущего пользователя. **$citizen**, в свою очередь определяет идентификатор пользоователя во время выполнения контракта. В данном случае, разрешение на модификацию колонки аватар получает только гражданин создавший ее.
 
