# Simple Kafka Connector 1C

Внешняя компонента, адаптер к 1С, позволяющий писать а так же получать сообщения из топиков Apache Kafka.

## Использование

### Подключение внешней компоненты

Можно подключать как изолировано (в child процессе), так и в рамках основного процесса 

`пример подключения на клиенте`
```1c
	Подключено = Ждать ПодключитьВнешнююКомпонентуАсинх("ОбщийМакет.Компонента", "Integration",, ТипПодключенияВнешнейКомпоненты.Изолированно);
	
	Если Не Подключено Тогда
	    Ждать УстановитьВнешнююКомпонентуАсинх(
	        "ОбщийМакет.Компонента");
	    Подключено = Ждать ПодключитьВнешнююКомпонентуАсинх(
	        "ОбщийМакет.Компонента", "Integration");		
		ПоказатьПредупреждение(Неопределено, СтрШаблон("Результат подключения: %1", ?(Подключено, "подключено", "ошибка подключения!")));
	КонецЕсли;
	
	Попытка
		Компонента = Новый("AddIn.Integration.simpleKafka1C");     
	Исключение
		ПоказатьПредупреждение(Неопределено, "Компонента <Simple Kafka 1C> не подключена!");
		Возврат;
	КонецПопытки;
```

`пример подключения на сервере`
```1c
	ПодключитьВнешнююКомпоненту("ОбщийМакет.Компонента", "Integration", ТипВнешнейКомпоненты.Native, ТипПодключенияВнешнейКомпоненты.Изолированно);		

	Попытка
		Компонента = Новый("AddIn.Integration.simpleKafka1C");     
	Исключение
		ВызватьИсключение "Компонента <Simple Kafka 1C> не подключена!";
	КонецПопытки;       
```

### Публикация сообщений 

```1c
	// инициализируем подключение к брокеру и топику  
	// Parametr1 - Брокеры, к которым производится подключение (строка) 
	// Parametr2 - Имя топика, в который производится публикация (строка) 
	Результат = Ждать Компонента.ИнициализироватьПродюсераАсинх(Брокеры, Топик);

	Если Результат.Значение Тогда
		Для т = 1 по КоличествоСообщений Цикл    
			// Parametr1 - Тело сообщения (строка)
			// Parametr2 - Номер партиции, по умолчанию = -1 (число)
			// Parametr3 - Произвольный ключ, идентифицирующий сообщение, например GUID
			// Parametr4 - Заголовки (строка), "ключ1,значение1;ключ2,значение2"
			РезультатПубликации = Ждать Компонента.ОтправитьСообщениеАсинх(СообщениеВКафку,, Ключ, Заголовки);  
			
			Если НЕ РезультатПубликации.Значение Тогда 
	        	Прервать; // здесь обработать ошибку бы надо...
			КонецЕсли;
			
		КонецЦикла;
		
		// освобождаем память
		// если не вызвать, то очень вероятна потеря части сообщений и расход памяти
		Ждать Компонента.ОстановитьПродюсераАсинх();  

	КонецЕсли;           
	
	Компонента = Неопределено;   
```

### Чтение сообщений

`пример чтения на клиенте`
```1c
    // указываем параметр - идентификатор группы, в рамках которой мы подписываемся на топик
	Ждать КомпонентаСлушателя.УстановитьПараметрАсинх("group.id", "odin_c");
	ПоведениеЧтенияСообщений = "OFFSET_STORED";	// читаем с последнего не прочитанного в топике
	Результат = Ждать КомпонентаСлушателя.ИнициализироватьКонсьюмераАсинх(Брокеры, Топик, ПоведениеЧтенияСообщений, 0);    
	ПодключитьОбработчикОжидания("СлушатьСообщения", 1);   
```

```1c
&НаКлиенте
Асинх Процедура СлушатьСообщения()          
	Смещение  = Неопределено;
	Сообщение = Неопределено;
	Ключ      = Неопределено;
	
	Сообщение = Ждать КомпонентаСлушателя.СлушатьАсинх(ТаймаутПолучения);     
	
	Если ЗначениеЗаполнено(Сообщение.Значение) Тогда       

    	ОбъектЧтение = Новый ЧтениеJSON;
		ОбъектЧтение.УстановитьСтроку(Сообщение.Значение);
		СтруктураJSON = ПрочитатьJSON(ОбъектЧтение);
		ОбъектЧтение.Закрыть();
		
		СтруктураJSON.Свойство("offset",  Смещение);
		СтруктураJSON.Свойство("message", Сообщение);	
		СтруктураJSON.Свойство("key",     Ключ);
		
	КонецЕсли;

КонецПроцедуры
```

## Поддерживаемые платформы

✔ Windows  
✔ Linux  