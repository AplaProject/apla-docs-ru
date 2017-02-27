################################################################################
Приложения
################################################################################
Приложение платформы eGaaS – это атономное программное решение для выполнения определенного действия или множества действий в рамках некоторой деятельности. Приложение состоит из 

* совокупность контрактов, реализующих его функционал, 
* таблиц базы данных и 
* шаблонов страниц и меню, обеспечивающих ввод и отображение данных. 

Структура приложений не является замкнутой и подразумевает возможность расширения за счет добавления новых контрактов, страниц и таблиц. 
 
Написание приложений осуществляется на программном клиенте eGaaS, который обеспечивает:

* открытие новых таблиц и добавление к ним необходимого количества колонок (пункт меню платформы **Tables**);
* создание новых контрактов (пункт меню **Smart contracts**);
* создание новых шаблонов страниц и меню (пункт меню **Interface**);
* добавление и редактирование параметров настроечной таблицы (ссылка *State parameters* на странице **Tables**);
* добавление и редактирование условий прав доступа к изменению таблиц, колонок таблиц, контрактов, страниц и меню, параметров настроечной таблицы.

Для написания контрактов и страниц шаблонов используются специальные языки платформы eGaaS.
 
При создании колонок таблиц приложений поддерживаются следующие форматы:

* Text – текстовый (без поддержки индекса);
* Numbers – целочисленные значения bigint	8 bytes;
* Date/Time – дата и время в формате  YYYY-MM-DD HH:MM:SS;
* Varchar(32) – строка длиной не более 32 символов;
* Money – целочисленное значение numeric;
* Double – число с плавающей точкой двойной точности.


********************************************************************************
Пример приложения
********************************************************************************
Рассмотрим пример создания простого приложения, обеспечивающего: 

* открытие  Центральным банком счетов граждан; 
* пополнение счетов Центральным банком;
* перевод денег между счетами. 

Первым делом создадим таблицу **accounts** с колонками: 

* **id** – номер счета, тип *Numbers + Index*; 
* **amount** – количество денег на счету, тип *Money + Index*;
* **citizen_id** – id гражданина, тип *Numbers + Index*.

На следующем шаге создаем контракты, необходимые для реализации каждого из действий:

**Открытие нового счета**

.. code:: js

	contract AddCitizenAccount {
	data {
		CitizenId string
	}
	func conditions {
	    
	    	$citizen_id = AddressToId($CitizenId)
		if $citizen_id == 0 {
			warning "not valid citizen id"
		}
		
		CentralBankConditions()
	
	}
	func action {
		
		DBInsert(Table("accounts"), "citizen_id", $citizen_id)
		}
	} 


В секции conditions происходит проверка id гражданина и наличие прав на открытие счета у пользователя вызывающего  этот контракт.  Для этого создается специальный контракт CentralBankConditions.

**Право подписывать контракты от имени Центрабанка**

.. code:: js

	contract CentralBankConditions {
	data {	}
	func conditions	{
	   if !IsGovAccount($citizen)
	   {
	       	warning "You have no right to this action"
	   }
	}
	func action {	}
	}

Сейчас в этом контракте правом совершать действия от имени Центробанка наделяется «создатель государства». В дальнейшем путем изменения этого контракта права подписи могут быть переданы гражданину, занимающего соответствующую должность в банке. Этот контракт в данном приложении выполняет роль смарт-закона, права на изменение которого могут принадлежать некоторому  представительному органу.

**Пополнение  счета**

.. code:: js

	contract RechargeAccount {
	data {
		AccountId int
		Amount money
		}
	
	func conditions	{
		CentralBankConditions()
		}

	func action {
		var recipient_amount money
            	recipient_amount = DBAmount(Table("accounts"), "id", $AccountId)
            	recipient_amount = recipient_amount + $Amount
            	DBUpdate(Table("accounts"), $AccountId, "amount", recipient_amount)
		}
	}

В качестве входных данных в контракте указываются номер счета гражданина и начисляемое количество денег. В секции  conditions проверяется права лица вызывающего этот контракт действовать от имени Центрабанка. В секции action реализуется сама процедура пополнения счета.

**Системный контракт перевода денег со счета на  счет**

Отдельный системный контракт перевода денег необходим прежде всего для того, чтобы предотвратить несанкционированный доступ к счетам. Именно он указывается в списке контрактов, имеющих право  изменять значение колонки *amount* таблицы **accounts**. Для этого при редактировании таблицы необходимо в поле *Permissions* у параметра *amount* вписать функцию *ContractAccess("MoneyTransfer","RechargeAccount")*.  После чего только эти два контракта будут иметь доступ к изменению счетов,  и транзакции между счетами во всех приложениях должны будут реализовываться только с помощью вызова контракта MoneyTransfer.

Системный контракт необходим также для того, чтобы предотвратить несанкционированное списание денег со счета пользователя путем использования скрытых вложенных контрактов. Для этого в системном контракте перевода денег используется механизм проверки подписи, описанный в разделе «Контракты с подписью».

.. code:: js

	contract MoneyTransfer {
	data {
		Amount money
		SenderAccountId int
		RecipientAccountId int
		Signature string "optional hidden"
		}
	func conditions {
    
	    	 if DBAmount(Table("accounts"), "id", $SenderAccountId) < $Amount {
			warning "Not enough money"
	    	 }
		}
	func action {

		    var sender_amount money
		    sender_amount = DBAmount(Table("accounts"), "id", $SenderAccountId)
		    sender_amount = sender_amount - $Amount
		    DBUpdate(Table("accounts"), $SenderAccountId, "amount",  sender_amount)

		    var recipient_amount money
		    recipient_amount = DBAmount(Table("accounts"), "id", $RecipientAccountId)
		    recipient_amount = recipient_amount + $Amount
		    DBUpdate(Table("accounts"), $RecipientAccountId, "amount", recipient_amount)

		}
	}

В контракте вставлена строка Signature string "optional hidden", вызывающая окно подтверждение транзакции (подробнее см. «Контракты с подписью»). В секции * conditions * производится проверка наличия достаточного количества денег на счету. 

**Пользовательский контракт перевода денег со счета на  счет**

Это основной контракт приложения реализующий перевод денег с вызовом системного контракта MoneyTransfer.

.. code:: js

	contract SendMoney {
	data {
		RecipientAccountId int 
		Amount money
		Signature string "signature:MoneyTransfer"
		}
	func conditions {}
	func action {
		MoneyTransfer("SenderAccountId,RecipientAccountId,Amount,Signature",$sender_id,$RecipientAccountId,$Amount,$Signature)
	}

Для созданных контрактов (кроме MoneyTransfer и CentralBankConditions, которые используются как вложенные) требуется создать интерфейсные формы для вода данных и вызова контракта. 

Прежде всего создадим новую  страницу Центрального Банка: позиция меню *Interface* программного агента eGaaS, далее кнопка addPage. Введем имя *CentralBank*, необходимые элементы навигации и две панели для вызова контрактов:

.. code:: js

	Title : Central bank
	Navigation( LiTemplate(government, Government),Central bank)
	MarkDown: ## Accounts 

	Divs(md-4, panel panel-default panel-body data-sweet-alert)
	    Form()
		Legend(" ", "Add citizen account")

		Divs(form-group)
		    Label("Citizen ID")
		    InputAddress(CitizenId, "form-control input-lg m-b")
		DivsEnd:

		TxButton{ Contract: AddCitizenAccount, Name: Add, OnSuccess: "template, CentralBank" }
	    FormEnd:
	DivsEnd:

	Divs(md-4, panel panel-default panel-body data-sweet-alert)
	    Form()
		Legend(" ", "Recharge Account")

		Divs(form-group)
		    Label("Account ID")
		    Select(AccountId, #state_id#_accounts.id, "form-control input-lg m-b")
		DivsEnd:

		Divs(form-group)
		    Label("Amount")
		    InputMoney(Amount, "form-control input-lg")
		DivsEnd:

		TxButton{ Contract: RechargeAccount, Name: Change, OnSuccess: "template,CentralBank" }
	    FormEnd:
	DivsEnd:

	PageEnd:


Здесь следует обратить внимание на то, что функция TxButton вызывая контракт автоматически передает в него значения полей формы если их id совпадают с именами входных параметров контрактов (CitizenId для контракта AddCitizenAccount и AccountId, Amount для контракта RechargeAccount).

Для доступа к созданной странице CentralBank необходимо добавить пункт в существующее меню, например, *government*: переходим к редактированию меню (со страницы Interface или из редактиро страницы CentralBank и добавляем в меню строку 

.. code:: js

MenuItem(CentralBank, load_template, CentralBank)

Также в редакторе страницы CentralBank необходимо указать меню, которое будет отражаться при открытии страницы Центробанка (разворачивающийся список *Menu*) – в нашем случае это меню *government*.

Осталось только открыть для редактирования страницу гражданина dashboard_default  и добавить к ней две панели для отражения номера счета и баланса и панель для вызова контракта перевода денег:

.. code:: js

	Divs(md-6)
	     Divs()
	     WiBalance(GetOne(amount, #state_id#_accounts, "citizen_id", #citizen#), StateValue(currency_name) )
	     DivsEnd:
	     Divs()
	     WiAccount( GetOne(id, #state_id#_accounts, "citizen_id", #citizen#) )
	     DivsEnd:
	  DivsEnd:


	 Divs(md-6, panel panel-default panel-body data-sweet-alert)
	    Form()
		Legend(" ", "Send Money")

		Divs(form-group)
		    Label("Account ID")
		    Select(RecipientAccountId, #state_id#_accounts.id, "form-control  m-b")
		DivsEnd:

		Divs(form-group)
		    Label("Amount")
		    InputMoney(Amount, "form-control")
		DivsEnd:

		TxButton{ Contract: SendMoney, OnSuccess: "template,dashboard_default,global:0" }
	    FormEnd:
	DivsEnd:

Теперь если у вас есть права прописанные в смарт-законе CentralBankConditions, то вы можете на странице Central bank открыть гражданам счета и пополнить их некоторыми суммами. После чего граждане смогут выполнять операции перевода денег со счета на счет.

