### Пример припложения

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

