---
title: "Часть 3"
translation: "extending-modx/getting-started/tutorial/part-3"
note: 'Некоторые подробности о коннекторах и процессорах относятся к MODX раньше 2.3.'
---

В MODX обработка форм осуществляется процессорами, которые представляют собой изолированные файлы, расположенные в основном каталоге MODX. Доступ к ним осуществляется через «коннекторы», которые обрабатывают запросы AJAX из пользовательского интерфейса (UI), для которых требуется переменная REQUEST с именем «action», указывающая, какой процессор отправлять. Процессоры отправляют очищенные данные REQUEST, а затем по окончании отвечают сообщением JSON обратно в браузер. Это позволяет быстро и легко выполнять запросы, снижающие нагрузку на сервер и браузер. В этом методе вы также можете сделать несколько асинхронных запросов к процессорам.

Вы можете думать о процессорах как модели в среде MVC. У MODX есть «модель», но в ней преобладают классы, которые обрабатывают слой ORM - другими словами, это не традиционный уровень модели. Процессоры часто постукивают, чтобы заполнить пробел.

Мы подробно рассмотрим процессор для создания чанка и покажем, как работают процессоры MODX.

Прежде всего, давайте предположим, что мы отправляем следующие данные в массив POST в соединитель, в котором переменная REQUEST «action» установлена ​​в «create», загружая соответствующую переменную create.php. В JS разъем `MODX.config.connectors_url+'element/chunk.php`, который разрешает (в нашей настройке по умолчанию):

> /modx/connectors/element/chunk.php

Оттуда коннектор проверит запрос, а затем отправит его соответствующему процессору по адресу:

> /modx/core/model/modx/processors/element/chunk/create.php

А теперь перейдем к процессору:

``` php
<?php
/**
 * @package modx
 * @subpackage processors.element.chunk
 */
$modx->lexicon->load('chunk');
```

Прежде всего, мы включаем корневой файл index.php для процессоров, который выполняет небольшую проверку переменных и включает лицензирование. Затем мы загружаем соответствующую тему лексики. В MODX Revolution языковые файлы i18n разделены на более мелкие файлы по темам (ранее называвшиеся фокусами). Здесь мы хотим, чтобы все языковые строки были в теме 'chunk'. Это экономит вычислительную мощность, загружая только соответствующие строки i18n.

**О темах**
Лексика _topics_ похожа на то, как популярный [gettext](http://www.gnu.org/software/gettext/) в структуре перевода _контекстов_ различает значения и предоставляет подмножества файлов перевода. Мы упоминаем об этом только для новичков, которые могут быть знакомы с системами, использующими gettext (например, WordPress): помните, что контексты в MODX сильно отличаются.

``` php
if (!$modx->hasPermission('new_chunk')) $modx->error->failure($modx->lexicon('permission_denied'));
```

Это проверка, чтобы убедиться, что пользователь имеет правильные разрешения для запуска этого процессора. Если нет, то он отправляет ответ об ошибке обратно в браузер через `$modx->error->failure()`. Ответ представляет собой строковое сообщение, переведенное через лексикон.

``` php
// значение по умолчанию
if ($_POST['name'] == '') $_POST['name'] = $modx->lexicon('chunk_untitled');

// избавиться от недействительных символов
$_POST['name'] = str_replace('>','',$_POST['name']);
$_POST['name'] = str_replace('<','',$_POST['name']);

// если имя для этого чанка уже существует, отправьте сообщение об ошибке
$name_exists = $modx->getObject('modChunk',array('name' => $_POST['name']));
if ($name_exists != null) return $modx->error->failure($modx->lexicon('chunk_err_exists_name'));
```

Теперь обратите внимание, как мы очищаем переменные и проверяем, нет ли уже чанка с этим именем.

``` php
// категория
$category = $modx->getObject('modCategory',array('id' => $_POST['category']));
if ($category == null) {
    $category = $modx->newObject('modCategory');
    if (empty($_POST['category'])) {
            $category->set('id',0);
    } else {
            $category->set('category',$_POST['category']);
            $category->save();
        }
}
```

Хорошо, здесь мы разрешаем создание динамической категории. Если указанная категория существует, она будет позже присвоена этой категории. Если нет, то он создает категорию в базе данных и подготавливает ее для последующего подключения к чанку.

``` php
// вызвать событие OnBeforeChunkFormSave
$modx->invokeEvent('OnBeforeChunkFormSave',array(
    'mode'  => modSystemEvent::MODE_NEW,
    'id'    => $_POST['id'],
));
```

События в Revolution почти такие же, как и в 096, но они более оптимизированы в своей загрузке.

``` php
$chunk = $modx->newObject('modChunk', $_POST);
$chunk->set('locked',isset($_POST['locked']));
$chunk->set('snippet',$_POST['chunk']);
$chunk->set('category',$category->get('id'));
if ($chunk->save() === false) {
    return $modx->error->failure($modx->lexicon('chunk_err_save'));
}
```

Важно: обратите внимание на 2-й параметр метода `newObject()`. В основном это то же самое, что и `$obj->fromArray()` - оно позволяет вам указать массив пар ключ-значение для назначения новому объекту.

``` php
// вызвать событие OnChunkFormSave
$modx->invokeEvent('OnChunkFormSave',array(
   'mode' => modSystemEvent::MODE_NEW,
   'id' => $chunk->get('id'),
));
```

Опять же, больше событий.

``` php
// действие диспетчера журналов
$modx->logManagerAction('chunk_create','modChunk',$chunk->get('id'));
```

Теперь, как действия менеджера работают в Revolution, немного по-другому. Здесь хранится лексиконный строковый ключ (`chunk_create`), ключ класса изменяемого объекта и фактический идентификатор объекта. Это позволяет более подробные отчеты о действиях менеджера.

``` php
$cacheManager= $modx->getCacheManager();
$cacheManager->clearCache();
```

Давайте просто и легко очистим кеш. Довольно легко, да?

``` php
return $modx->error->success('',$chunk->get(array('id', 'name', 'description', 'locked', 'category')));
```

Теперь отправьте успешный ответ обратно в браузер. Параметры `$modx->error->success()` следующие:

1: $message - Строковое сообщение для отправки назад. Используется для сообщения подробностей об успехе (или неудаче).
2: $object - `xPDOObject` или массив полей данных для преобразования в JSON и отправки обратно в браузер.

Итак, в основном, здесь, мы отправляем обратно информацию о чанке - за исключением контента, который может быть большим, ненужным и сложным для отправки. Это позволит пользовательскому интерфейсу правильно обрабатывать создание.

Далее мы поговорим о том, как создавать свои собственные схемы и динамически добавлять их в инфраструктуру MODX без необходимости изменять ядро.
