# Performance

Не знаю как всю эту информацию систематизировать.  
И да результаты варьируются от раза к разу, но я буду ими манипулировать, 
так как разница в плюс минус небольшая (с моего устройства MacBook Air).

**Основные показатели:**  
DOM загрузился за **1.85s**, а вот полностью только за **7.80s**  
Быстренько смотрим Audit  
Первое значимое отображение (First Meaningful Paint) **5.6ms**  
Первая интерактивность (First Interactive) **33.540ms**  
Показатели прямо таки не ахти. 

Выяcняем почему.

**Render Blocking Scripts & Stylesheets** 

В head страницы просто немереное количество скриптов и линков.
- Самое простое - перенести все скрипты к закрывающемуся тегу body 
- Нас это не устраивает, ведь мы знаем, что есть async и defer (между прочем, у каких-то скриптов там проставлены эти значения, причем даже примиксованы друг с дургом)  
Я бы выставляла по такому приоритету:  
с async - скрипты от которых ничего не зависит и к закрывающему тегу body их, в head остаются скрипты, только те, 
которые должны быть загружены перед тем как загрузится дом (интересно какие это) и 
с defer, которые должны загрузится первыми и запуститься, как только DOM загрузится. Так мы не будем блокировать рендеринг. 
Но как было сказано на лекции, это надо тестить и проверять как все это дело себя будет вести.
- Современный вариант - Resource Hints. Мы можем использовать preload, Чтобы предварительно загрузить ресурсы. 
При этом мы можем использовать их не только для скриптов. Например, можно использовать их как для стилей, изображений и шрифтов и поддержка уже почти везде (opera mini и IE  - они там всегда).  
Так же можно использовать prefetch для ресурсов с низким приоритетом, которые пригодятся позже и сохранятся в кеше.

Т.е сразу можно сделать preload нужного нам шрифта. И кстати о шрифте, 
(при пост просмотре обнаружила, что у него в headers стоит link: <https://fonts.gstatic.com>; rel=preconnect; crossorigin)   
https://fonts.googleapis.com/css?family=Roboto+Condensed:400,700|Roboto:300,300i,400,400i,500,500i,700,900&subset=cyrillic  
Это просто какой-то гиганстский набор для первой загрузки, при этом если заглянуть внутрь, как я понимаю загружаются шрифты не только кириллические, а весь дружный набор
(cyrillic-ext, cyrillic, greek-ext, greek, vietnamese, latin-ext, latin). А для первой отрисовки, да и вообще для всей главной страницы, нужны всего два начертания: 
Roboto c font-weight: normal и condensed font-weight: bold для заголовков на карточках. Т.е просто сокращаем link наши шрифты rel="preload" as="font" type="crossorigin";
Так можно поступить с логотипом и может заглушкой для неавторизованного пользователя, но не могу сказать насколько это целесообразно. 
Так же что-то можно выставить в заголовках и даже использовать в media выражения, но нам здесь это не поможет. 
Весь критический css тоже ставим под preload с ` as="style" onload="this.rel='stylesheet'`

Ну вообще первыми на страницу подгружаются всякие аналитические скрипты, 
скрипты реклам и для этого нашла такое решиние, асинхронно и не блокирует window onload т.е постройку DOM  
`<link rel="preload" as="script" href="async_script.js" onload="var script = document.createElement('script'); script.src = this.href; document.body.appendChild(script);">`

Где можно использовать технику прогрессивного css (вставляя link перед каждым блоком), я не вижу, на главной странице, все очень похоже.

Если посмотреть во вкладку Network  
![Order](./src/images/order.png)
Видим что  ресурсы встали в очередь одновременно с одинаковым приоритетом, а загружаются в разное время.

Я бы сказала, что это ограничение по протоколу HTTP на количество паралельных запросов, 
но вроде все эти ресурсы идут по протоколу HTTP 2 (где количество паралельных запросов больше) и используют cdn сети, но что-то пошло не так.

**Optimize (Minify) resources** 

Банально, но многие скрипты и css файлы не минифицированы, так же дело обстоит и с изображениями, а их количество доминирует
![Order](./src/images/optimaze.png)

**Save images in next-gen formats** 

Использовать прогрессивный JPEG, WepP, Lossy GIF, SVGOMG

Плюс сюда можно добавить, что как раз таки изображения грузятся по HTTP 1

**Unused CSS rules & JS** 

Грузитсся очень достаточно большое количество скриптов и css файлов, которые не используются в документе. 
Например, fontawesome, outer.css, outer.js, main.css, main.min.css, all.min.css?ver=1.0.0, style.css?ver=14, nivo-lightbox-overlay min.css, deafult.css,
common.js, viewed.js, all.min.js?ver=1.0.0, nivo-light.js, 
front.js (тоже не нужен на этой странице и судя по содержанию этого скрипта, я бы им посоветовала все таки определиться с одним слайдером, максиму двумя, а то там case на все случаи жизни)

Думаю под этот пункт так же можно записать, что существуют такие файлы как homepage.css?ver=7, в которых всего 3 строчки, почему, зачем, за что?
 Их либо размещать инлайн, либо добавить уже в какой нибудт common.css
 
 
**Layout**

Макет сверстан на float, вроде бы как flex быстрее будет

Происходит перерисовка макета - плашка - клуб лайфхакеров - заменить top на transform  и сделать will-change: transform (для старичков transform: translateZ(0))
Для самих изображений скорее не надо выделять, так как будет выделять память и утекать.

И вот мы дошли до этих изображений, которые якобы загружаются c lazyloading  и должны вообще опитимизировать загрузку страницы, но как раз таки эти скрипты попадают
под профилирование как LONG TASK (остальное почти все это аналитеские скрипты).
![Order](./src/images/isElementInViewPort.png)
![Order](./src/images/lazyloadTumbanail.png)

Получается, что функция `lazyLoadThumbnails` расчитывается почти всегда по скролу, почему? потому что вот, что проверяется `if ($self.css('opacity') == 1) return;`,
и если наша мышь находится на картинках и мы проскроливаем, мы снова выполняем скрипт и перасчитываем, потому что при скроле opacity меняется на 0.8 
(т.е либо > 0, !==0, либо навешивать класс, потому что при ховере стоит !important это уж совсем некрасиво)
а isElementInViewport выполняется всегда при скрорле, а там `(rect.top - 300) <= (window.innerHeight || document.documentElement.clientHeight)` 
Вообще все эти height можно сохранить куда-то выше и только перарасчитывать при рейсайзе, а при показе всех изображений отвязывать событие scroll вообще.
Долгое сохранение в объект $self.

Решение - Как я поняла, сейчас уже можно использовать для такого **IntersectionObserver API** чтобы следить попал ли элемент в область видимости

Вообще тема написания js здесь довольно таки больная, есть что поправить

Этот кусочек заставляет дергаться страницу в конвульсиях после 7 секунд (более точно через то время, когда все элементы страницы загрузятся полностью )
`$(window).load(function () {
     $('.top-banner').show();
});`

Здесь определяется `var opacBgEl = '<div class="opac-bg" id="opac-bg"></div>';`, который потом добавляется в DOM (почему бы сразу его там не рендерить ?),
а потом через Show и hide показывается, только вот по вызову этих функций он постоянно ищется `$('body').find('#opac-bg').show();` 

Мобильное меню показывается и скрывается через ` $mobileMenu.fadeIn('fast');` и `$mobileMenu.fadeOut('fast');` соотвественно, все это можно сделать через 
transform и opacity насколько мне известно, чтобы не перерисовывать весь DOM

и там множество такого - 
` $(window).on('scroll', function () {
    addTestPixelUrl();
})`
где по скролу вызывается функция, в которой есть проверки `isVisible(testPixel) && typeof(testPixel.attr('url')) == 'undefined'`,
в первой вычисляет долистали ли мы этого элемента, но она выполняется всегда по скролу, а тап постоянно перерасчитывается window.height, 
хотя единожды поставив testPixel.attr('url') в какое либо значение, нам больше этого не надо, достаточно просто поменять уловие местами
`typeof(testPixel.attr('url')) == 'undefined' && isVisible(testPixel)` и перерачет не будет выполняться каждый раз, даже если мы уже выставили атрибут.

возможно это и не задание про рефакторингу кода, но в Performance частенько Forced Reflow, что чаще всего в этом случае из-за JS, т.е приходу к выводу,
что решением здесь как раз таки и будет рефакторинг кода.

**Cache static assets**

У некоторых статический файлов не проставлены max-age или expires

А у тех, которых проставлены, зачастую необоснованное использование: 4.2m для google-analytics  или nivo-lightbox кешируется на 24 часа,
а jquery на 7 дней, это конечно больше, но для ресурса, который будет неизменен.
Вывод нужно поработать над Response Headers

**Small things**

Все ссылочки с `http://` заменить на просто `//` - мелочи, а приятно

Часть верстки используется под мобильные устройства, часть под декстопныеб т.е в нем ненужный HTML - перверстать грамотно, используя одну разметку.

Использовать для изображений тэг picture по возможности, чтобы предоставлять контент разного качества (@2x, @3x) разным клиентам

Как я поняла к некоторым ресурсам лучше было не приминять было GZIP (Обращаем внимание на размер файлов)
вверху должен быть ужатый размер - Transferred size  
внизу - Actual size  
Можно указать минимальный размер  файла, потому что файлы весят мало и скорре всего на поддержку словаря GZIP уходит больше затрат, 
чем на его сжатие.
![Order](./src/images/gzipBigger.png)

П.С и это все результаты для PC, например, для мобильного устроства при скорости SLOW 3G DOM загружается за 11.8s ! а полностью - за 56.04s




 
 
