```js
	пРазмерПачки = 100000;
	пСтарт = ТекущаяДата();
	пСообщение = XMLСтрока(ТекущаяДата()) + ": Начинаю расчет количества записей...";
	Сообщить(пСообщение);
	Запрос = Новый Запрос;
	Запрос.Текст = 
	"ВЫБРАТЬ
	|	КОЛИЧЕСТВО(ОчередьАвтоотчетов.Соглашение) КАК КоличествоЗаписей
	|ПОМЕСТИТЬ тКоличествоЗаписей
	|ИЗ
	|	РегистрСведений.ОчередьАвтоотчетов КАК ОчередьАвтоотчетов
	|;
	|
	|////////////////////////////////////////////////////////////////////////////////
	|ВЫБРАТЬ
	|	ОчередьАвтоотчетов.ДатаКон КАК ДатаКон,
	|	ОчередьАвтоотчетов.ДатаНач КАК ДатаНач,
	|	ОчередьАвтоотчетов.ВидАвтоотчета КАК ВидАвтоотчета,
	|	ОчередьАвтоотчетов.Соглашение КАК Соглашение
	|ПОМЕСТИТЬ тКоличетвоДляОбработки
	|ИЗ
	|	РегистрСведений.ОчередьАвтоотчетов КАК ОчередьАвтоотчетов,
	|	тКоличествоЗаписей КАК тКоличествоЗаписей
	|ГДЕ
	|	ОчередьАвтоотчетов.ИдентификационныйНомер > тКоличествоЗаписей.КоличествоЗаписей
	|;
	|
	|////////////////////////////////////////////////////////////////////////////////
	|ВЫБРАТЬ РАЗЛИЧНЫЕ
	|	тКоличетвоДляОбработки.ДатаКон КАК ДатаКон,
	|	тКоличетвоДляОбработки.ДатаНач КАК ДатаНач,
	|	тКоличетвоДляОбработки.ВидАвтоотчета КАК ВидАвтоотчета,
	|	тКоличетвоДляОбработки.Соглашение КАК Соглашение
	|ИЗ
	|	тКоличетвоДляОбработки КАК тКоличетвоДляОбработки";
	
	РезультатЗапроса = Запрос.ВыполнитьПакет();
	РезультатПакет0 = РезультатЗапроса[1];
	РезультатПакет1 = РезультатЗапроса[2];
	Если НЕ РезультатПакет0.Пустой() Тогда 
		пКоличествоЗаписей = РезультатПакет0.Выгрузить()[0].Количество;
		пСообщение = СтрШаблон("%1: Количество записей для изменения составляет: %2",
                                      XMLСтрока(ТекущаяДата()), пКоличествоЗаписей);
		Сообщить(пСообщение);
		Выборка = РезультатПакет1.Выбрать();
		//инициализация общего счетчика обработки записей
		пСчетчик = 1;
		//инициализация счетчика уникальных номеров
		пИдентификационныйНомер = 1;
		пСообщение = XMLСтрока(ТекущаяДата()) + ": Начинаю изменение идентификационных номеров...";
		Сообщить(пСообщение);
		//инициализация таблицы протокола
		пПротокол = Новый ТаблицаЗначений;
		пПротокол.Колонки.Добавить("number_old");
		пПротокол.Колонки.Добавить("number_new");
		//инициализация счетчкика размера пачки транзакции
		пСчетчикПачки = 0;
		Пока Выборка.Следующий() Цикл	
			ОбработкаПрерыванияПользователя();
			Если НЕ ТранзакцияАктивна() Тогда 
				НачатьТранзакцию();
			КонецЕсли;
			//получение записей из РС по отбору
			Запись = РегистрыСведений.ОчередьАвтоотчетов.СоздатьНаборЗаписей();
			Запись.Отбор.Сбросить();
			Запись.Отбор.ДатаКон.Установить(Выборка.ДатаКон);
			Запись.Отбор.ДатаНач.Установить(Выборка.ДатаНач);
			Запись.Отбор.ВидАвтоотчета.Установить(Выборка.ВидАвтоотчета);
			Запись.Отбор.Соглашение.Установить(Выборка.Соглашение);
			Попытка
				Запись.Прочитать(); 
				Запись.ОбменДанными.Загрузка = Истина;
				//инициализация соответствия номеров
				пСоответствиеИдентификаторов = Новый Соответствие;
				//запись измененных данных
				Для Каждого пСтрока Из Запись Цикл
					пНовыйНомер = пИдентификационныйНомер;
					пПрошлыйНомер = пСтрока.ИдентификационныйНомер;
					пСоответствиеИдентификаторов.Вставить(пПрошлыйНомер, пНовыйНомер);
					пСтрока.ИдентификационныйНомер = пИдентификационныйНомер;
					//блок счетчики
					пСчетчик = пСчетчик + 1;
					пИдентификационныйНомер = пИдентификационныйНомер + 1;
					пСчетчикПачки = пСчетчикПачки + 1; 
					Если пСчетчик%100000 = 0 Тогда
						пСообщение = СтрШаблон("%1: Обработано записей: %2 из %3",
                                                                      XMLСтрока(ТекущаяДата()), пСчетчик, пКоличествоЗаписей);
						Сообщить(пСообщение);
					КонецЕсли;
				КонецЦикла;
				Запись.Записать(Истина);
				Если пСчетчикПачки >= пРазмерПачки Тогда 
					ЗафиксироватьТранзакцию();
					НачатьТранзакцию();
					пСчетчикПачки = 0;
				КонецЕсли;
				
			Исключение
				//при неудачной записи набора - откатываем нумерацию на количество набора
				пИдентификационныйНомер = пИдентификационныйНомер - пСчетчикПачки;
				//добавление записей в протокол
				Для Каждого пСтрока Из пСоответствиеИдентификаторов Цикл
					пНоваяСтрока = пПротокол.Добавить();
					пНоваяСтрока.number_old = пСтрока["Ключ"];
					пНоваяСтрока.number_new = пСтрока["Значение"];
				КонецЦикла;
				Сообщить("Произошла ошибка изменения записи: " 
				+ Символы.ПС
				+ ОписаниеОшибки());
				ОтменитьТранзакцию();
			КонецПопытки;
			
		КонецЦикла;
		пСообщение = XMLСтрока(ТекущаяДата()) + ": Перенумерация номеров отчетов завершена...";
		Сообщить(пСообщение);
		Если пПротокол.Количество() <> 0 Тогда 
			ПечатьТЗ(пПротокол, , , , "Протокол работы");
		КонецЕсли;
		
		Если ТранзакцияАктивна() Тогда 
			ЗафиксироватьТранзакцию();
		КонецЕсли;
	Иначе
		пСообщение = XMLСтрока(ТекущаяДата()) + "Результат запроса пустой!!!";
		Сообщить(пСообщение);
	КонецЕсли;
	пВремяВыполнения = ТекущаяДата() - пСтарт;
	пСообщение = "Период выполнения: " + XMLСтрока(пВремяВыполнения);
	Сообщить(пСообщение);
```
