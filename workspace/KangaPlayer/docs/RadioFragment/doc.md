
توثيق RadioFragment – KangaPlayer

1. نظرة عامة

RadioFragment هو الشاشة الرئيسية لقسم الراديو في تطبيق KangaPlayer. يعرض محطات الراديو بعد تصنيفها إلى قسمين:

· Best Live Now: أفضل 10 محطات (التي تم التحقق من عملها مؤخراً) من الدولة المختارة.
· All Stations: باقي المحطات لنفس الدولة.

يعتمد التصميم على MVVM مع فصل صارم للمسؤوليات (SRP) ويستخدم مكتبة radiobrowser4j لجلب البيانات من Radio Browser API. يتم تخزين البيانات محلياً باستخدام RadioCacheManager لتقليل استهلاك الشبكة وتسريع التحميل.

2. هيكلية الملفات

تنقسم الملفات إلى عدة طبقات:

الطبقة الملفات الوصف
UI RadioFragment.java الشاشة الرئيسية
ViewModels RadioViewModel.java حالة الشاشة ومنطق الأعمال
Models RadioUiState.java حاوية حالة UI
 StationPreview.java نموذج بيانات المحطة
 CountryItem.java نموذج الدولة
Repository RadioRepository.java جلب البيانات من API والتخزين المؤقت
 RadioCacheManager.java إدارة ذاكرة التخزين المؤقتة
Use Cases FetchCountriesUseCase.java جلب قائمة الدول
 FilterByCategoryUseCase.java تصفية المحطات حسب التصنيف
API RadioBrowserService.java إعداد الاتصال بـ RadioBrowser API
Utilities RadioConstants.java الثوابت
 Event.java غلاف أحداث LiveData
 RadioScrollHelper.java مساعد التمرير
Adapters BestLiveAdapter.java محول قسم Best Live
 AllStationsAdapter.java محول قسم All Stations
 HeaderTitleAdapter.java عناوين الأقسام
Dialogs CountrySelectionDialogFragment.java نافذة اختيار الدولة
 CategoryFilterBottomSheet.java نافذة تصفية التصنيفات
Application KangaPlayer.java Application class
 ConnectivityLiveData.java مراقبة الاتصال بالإنترنت

3. شرح الملفات والكلاسات

3.1 Application & Connectivity

KangaPlayer
يمثل Application class. يوفر:

· getContext(): السياق العام.
· getAppExecutor(): ThreadPool مشترك للعمليات الخلفية.
· getFileCacheManager(): مدير التخزين المؤقت.
· getConnectivityLiveData(): مراقب الاتصال بالإنترنت.

ConnectivityLiveData
LiveData<Boolean> تراقب حالة الاتصال بالإنترنت باستخدام ConnectivityManager.NetworkCallback (API ≥ 24) أو BroadcastReceiver (أقدم). تستخدم في RadioViewModel لمنع محاولات التحميل دون اتصال.

3.2 الثوابت (RadioConstants)

· PAGE_SIZE = 5000: الحد الأقصى لعدد المحطات المحملة.
· BEST_LIVE_LIMIT = 10: عدد محطات Best Live.
· STATIONS_CACHE_MAX_AGE_MILLIS: صلاحية الكاش (6 ساعات).

3.3 نماذج البيانات (Models)

StationPreview
يحتوي على: stationUuid, name, favicon, countryCode, language, codec, bitrate, hls, lastCheckOk.
يتم استخدامه لعرض المحطة في القوائم. لا يحتوي على بيانات ديناميكية ثقيلة.

CountryItem
يحتوي على: name, code, stationCount. قابل للـ Parcelable لتمريره بين الأجزاء.

3.4 Service (RadioBrowserService)

Singleton لإعداد كائن RadioBrowser بإعدادات الاتصال (timeout=8000، User-Agent). يستخدم EndpointDiscovery لاكتشاف أنسب خادم API.

3.5 Repository (RadioRepository)

يتعامل مع API و Cache:

· getCountries(forceRefresh): قائمة الدول.
· fetchStationPreviews(countryCode): تحميل المحطات مع الكاش.
· fetchStationPreviewsByTag(tag): محطات بتصنيف معين.
· getCategories(countryCode): استخراج التصنيفات من أول 100 محطة للدولة.
· toPreview(): تحويل كائن المكتبة إلى StationPreview.

استخدام الكاش:
دالة fetchStationPreviews تتحقق أولاً من RadioCacheManager.isCountryCacheValid(). إذا كان الكاش صالحاً، تُعاد البيانات المخزنة. وإلا تُجلب من الشبكة وتُخزّن.
يتم تخزين البيانات بصيغة JSON في مجلد json_cache باسم radio_cache_{countryCode}_previews.json.
ملف التعريف radio_cache_{countryCode}_meta.json يحتوي على lastUpdated و totalCount.

3.6 Cache Manager (RadioCacheManager)

يوفر:

· isCountryCacheValid(): فحص صلاحية الكاش.
· getStationPreviewPage() / saveStationPreviewPage(): قراءة/كتابة المحطات مع ملف meta.
· getCategories() / saveCategories(): تصنيفات الدولة.
· getCountries() / saveCountries(): قائمة الدول مع صلاحية مستقلة.
· وظائف لتتبع التصويت والنقر عبر SharedPreferences.

صلاحية الكاش:

· الدول: 6 أيام.
· المحطات: 6 ساعات.

الملفات:
radio_cache_{countryCode}_previews.json
radio_cache_{countryCode}_categories.json
radio_cache_{countryCode}_meta.json
countries.json

3.7 Use Cases

FetchCountriesUseCase: يجلب قائمة الدول مع إدارة الأخطاء (شبكة/غيرها) وتحديث واجهة المستخدم.

FilterByCategoryUseCase: يجلب المحطات بتصنيف معين، ويعيد تقسيمها إلى Best Live (مع حد أقصى 10) و All Stations.

3.8 ViewModel (RadioViewModel)

يدير حالة الشاشة عبر RadioUiState:

· initialize(): إذا كانت الدولة محفوظة، يحمل المحطات، وإلا يطلب اختيار الدولة.
· setCountry(): يحفظ الدولة في SharedPreferences، يمسح كاش الدولة السابقة، ثم يحمل البيانات.
· setCategory(): إذا null (All)، يعيد التحميل الكامل، وإلا يستخدم FilterByCategoryUseCase.
· loadAllStations(): يجلب التصنيفات والمحطات (مع الكاش) في خيط خلفي، ثم يقسم إلى Best Live و All Stations.
· extractBestLive(): يأخذ أول BEST_LIVE_LIMIT محطة بحالة lastCheckOk.
· extractAll(): يستبعد محطات Best Live من القائمة الكاملة.
· يراقب الاتصال: عند انقطاعه، يظهر رسالة بدون Retry. عند فشل التحميل (IOException) يظهر رسالة مع Retry. الأخطاء الأخرى بدون Retry.

LiveData:

· uiState: حالة UI (تحميل، خطأ، قوائم، فئات).
· showCountrySelectionEvent: حدث لمرة واحدة لإظهار نافذة اختيار الدولة.

3.9 UI State (RadioUiState)

يحمل:

· loading, errorMessage, errorRetryable
· bestLive, allStations
· currentCountryCode, currentCategory
· categories, countries

يستخدم Builder pattern لإنشاء غير قابل للتغيير.

3.10 Fragment (RadioFragment)

· ينشئ RadioViewModel ويربطه.
· يبني RecyclerView مع ConcatAdapter يضم Best Live ثم All Stations.
· يراقب uiState لتحديث القوائم وإظهار/إخفاء مؤشر التحميل.
· يعرض Snackbar مع زر Retry فقط عند isError && errorRetryable.
· يفتح CountrySelectionDialogFragment و CategoryFilterBottomSheet عند الحاجة.
· عند النقر على محطة، يفتح RadioPlayerActivity ويمرر stationUuid و name.
· يدعم التمرير للأعلى عبر RadioScrollHelper.
· يدير دورة حياة RadioImageLoader.

3.11 Adapters

BestLiveAdapter و AllStationsAdapter:

· يستخدمان ListAdapter مع DiffUtil لتحديث فعّال.
· يعرضان: صورة المحطة، الاسم، مؤشر HLS (بلون Primary إذا مدعوم، باهت إذا لا)، bitrate بصيغة xxx kbps.
· زر Play يمرر الضغط إلى OnStationActionListener.

HeaderTitleAdapter: يعرض عنوان القسم ("Best Live Now" أو "All Stations").

3.12 Dialogs

CountrySelectionDialogFragment:

· Modal Bottom Sheet بقائمة دول قابلة للبحث.
· يدعم التحديد المسبق للدولة المحفوظة (via preselectedCode).
· عند التأكيد، يستدعي listener.onCountrySelected().

CategoryFilterBottomSheet:

· مشابه لنافذة الدول لكن للتصنيفات مع خيار "All" دائم.
· يرجع التصنيف المختار أو null عند اختيار "All".

3.13 الأخطاء ومعالجتها

نوع الخطأ مثال Retry رسالة المستخدم
لا يوجد اتصال إنترنت ConnectivityLiveData = false لا "No internet connection..."
خطأ شبكة أثناء التحميل IOException, SocketTimeoutException نعم "Network error..."
خطأ غير متوقع Exception عامة لا "Failed to load..."

· جميع الرسائل قابلة للترجمة (موجودة في strings.xml).
· Snackbar تعرض رسالة مع زر Retry عند الحاجة.
· لا يستخدم التطبيق Toast نهائياً.

4. آلية العمل

1. بدء التشغيل:
      RadioFragment.onViewCreated() -> viewModel.initialize().
2. إذا لم يختر المستخدم دولة مسبقاً:
      يظهر حدث showCountrySelectionEvent -> تفتح نافذة اختيار الدولة.
      عند اختيار دولة، يحفظ الكود في SharedPreferences، ثم يستدعي loadAllStations().
3. تحميل المحطات:
   · يفحص الاتصال، وإذا كان مقطوعاً تظهر رسالة فورية.
   · في خيط خلفي: يجلب التصنيفات (أول 100 محطة) والمحطات (حتى 5000) مع الكاش.
   · يستخرج Best Live (أول 10 محطات عاملة) و All Stations (البقية).
   · يحدث واجهة المستخدم.
4. تصفية حسب التصنيف:
      عند اختيار تصنيف من bottom sheet، إذا كان غير "All"، يحمل المحطات من API بتصنيف محدد ويقسمها بنفس المنطق. عند اختيار "All"، يعيد التحميل الكامل للدولة.
5. النقر على محطة:
      يفتح RadioPlayerActivity مع EXTRA_STATION_UUID و EXTRA_STATION_NAME. المشغّل يستخدم UUID لجلب تفاصيل المحطة الكاملة وتشغيل الدفق.

5. علاقة RadioFragment بباقي التطبيق

· KangaPlayer (Application): يوفر ConnectivityLiveData, FileCacheManager, ExecutorService.
· RadioPlayerActivity: يستقبل UUID المحطة ويشغلها.
· MainActivity / HomeActivity: يستضيف RadioFragment ضمن Bottom Navigation.
· DebugActivity: يعرض الأخطاء غير المتوقعة عبر UncaughtExceptionHandler.
· TopLevelDestinationActions و Filterable واجهات تنفذها RadioFragment لتتكامل مع الـ Toolbar/BottomNavigation.

6. ملاحظات إضافية

· البيانات: لا توجد بيانات ديناميكية ثقيلة. كل البيانات تعرض لمرة واحدة عند التحميل. رقم النقرات (clickCount) أزيل نهائياً لتخفيف الحمل.
· الكاش: يعمل بشفافية. عند العودة لدولة سبق تحميلها خلال 6 ساعات، تظهر البيانات فوراً من الملفات المخزنة دون اتصال بالشبكة.
· الأداء: الحد الأقصى 5000 محطة، مع تحميلها دفعة واحدة. هذا مقبول لأغلب الدول. الولايات المتحدة قد تحتوي ~6000 محطة، وسيتم تجاهل البعض بسبب PAGE_SIZE. يمكن زيادته إذا لزم الأمر.

