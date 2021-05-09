
#MRedux跨平台框架设计

##简介

MRedux全称是mobile-redux，即移动端的redux（redux是前端的数据管理框架），这次我们借鉴了前端redux的思想，构建了一个业务框架，汲取了redux单向数据流，数据驱动ui等优秀的特性。这是一个和mvp/mvvm并行的框架，相比起命令式的mvp，响应式的MRedux可以更好的隔离ui和业务的逻辑，而实现mvvm有多种方式，比起android的databinding，MRedux则更注重跨平台，采用纯kotlin编写，既可直接用于android，也可以用于kotlin-native跨平台使用



##目标

react响应式设计，把ui抽象成数据结构(State)，逻辑层只改动ui对应的数据，ui自动变化
跨平台设计，业务逻辑以及数据可共用（kotlin-native），android/ios分别监听数据变化更新ui
单向数据流管理及记录


##架构图
![avatar](https://note.youdao.com/yws/public/resource/d176f3a15c61f3c52a6375aea2d85202/xmlnote/FC774928665A4CED9480622A90D63E18/7582)

State表示当前页面所有ui元素对应字段的数据结构抽象，构建出State，并且用view去subscribe对应数据的变化，主要做data到view的绑定
用户发出action，一个action表示用户行为，比如进入页面发起获取data的action，点一个按钮发起一些逻辑操作的action，action主要来源于view层的操作
action被dispatch，经过一系列扩展的middleware到达图上3的位置进行逻辑处理并改变State，（如果一些action需要被共享，可以放到reducer处理），最后State触发ui更新


##State

state是ui元素的数据抽象，比如一个界面有title，中间有feed列表，我们可以做这样的抽象

data class FeedItem(

    val _id: String = "",

    val icon: String = "",

    val content: String = ""

): State(_id)



data class AllFeedState(

    val datas: List<FeedItem> = emptyList()

) : State("")



val loading: LiveState<BoolState> = BoolState(value = false).toLiveState()

val title: LiveState<StringState> = StringState(value = "default").toLiveState()

val feedList: LiveState<AllFeedState> = AllFeedState().toLiveState()

上面3个LiveState就是对整个ui的抽象（一个title，一个列表, 一个loadingView），我们需要把每个字段都用LiveState包裹用来让每个字段可以单独被观测，简而言之，哪个ui元素需要被独立观察，哪个字段就应该被放到一个LiveState里面（像loading这类通用的页面状态我们已经集成到了Store中, 直接观察store.commonState并使用自带的LoadingAction, ErrorAction, EmptyAction 就能收到通知）



##LifecycleStore

store是整个框架的核心，它持有界面的ui数据State，同时能够dispatch action，还能够添加middleware，而LifecycleStore实现了Store接口，加入了界面生命周期的概念。所以对于android来说，可以是一个activity对应一个store，也可以是一个fragment对应一个store，对于ios，一个viewController则对应一个store，界面退出，相关数据就会被自动释放。所以对于我们的例子，可以新建一个FeedStore。我们推荐使用BaseLifecycleStore作为您的Base类，它集成了更多功能

class FeedStore(

// 注意此处需要默认值

    context: Any = Any(),

    //repo是数据层，可以是网络或者数据库组件

    val repo: Repo 

): BaseLifecycleStore<Any>(

    context

){

val loading: LiveState<BoolState> = BoolState(value = false).toLiveState()

val title: LiveState<StringState> = StringState(value = "default").toLiveState()

val feedList: LiveState<AllFeedState> = AllFeedState().toLiveState()

}

这里我们把刚才定义的State作为属性放到store里面。LifecycleStore的构造函数有两个字段，一个是初始化的context（注意此处需要默认值），需要带进store的上下文信息，注意此处不是android的context，可以是任何东西，不过不需要context传入Unit也可以。第二个是middleware列表，用户可以自定义middleware传入store中，middleware是一个拦截器，后面会说。现在我们有一个Store了，它知道对应的ui界面长什么样子了。



现在是时候把ui和数据关联起来的了

class FeedActivity: Activity(){

    private val store: FeedStore = createLifecycleBindingStore()

    

    fun onCreate(){

        val titleView = findViewById<TextView>(R.id.title)

        store.title.subscribe(store){ title -> 

            titleView.text = title

        } 



        store.feedList.subscribe(store){ feed: AllFeedState -> 

            val dataList = feed.datas;

            //recyclerView notifyDataChanged

        } 

       

    }

}

调用createLifecycleBindingStore， 会生成一个对应的业务store， 比如我们的FeedStore，只要在前面声明类型，就会自动创建并且绑定生命周期，它其实有几个默认参数，可以在特殊情况下自定义Store的创建，我们不建议直接new创建Store，如果需要的话，可以new FeedStore，然后调用bindLifecycleToStore方法去绑定生命周期



##Action

现在我们已经创建好了Store，并且把它和View关联起来了，目前看只要State变化了，ui就会收到通知。现在我们可以创建Action来加载数据

data class LoadingAction(val loading: Boolean): Action<FeedStore>{

    override fun run(store: FeedStore) {

            store.loading.state.copy(value = loading).notifyChange()

    }

}



data class FeedFetchDataAction(val requestId: String): Action<FeedStore>{

    override fun run(store: FeedStore) {

        store.dispatch(LoadingAction(true))

        val retrofit = store.repo.retrofit

        retrofit.api(requestId, { rawList ->

          //success   

          store.dispatch(LoadingAction(false))

          store.title.state.copy(title = "xxx").notifyChange()



          //如果需要的话，后台返回数据解析成我们需要的State的格式

          val feeds = process(rawList)

          store.feedList.state.copy(datas = feeds).notifyChange()

        }, {

          //failed

          store.dispatch(LoadingAction(false))

        })

    }

}

我们可以在上面创建的activity的onResume中调用

store.dispatch(FeedFetchDataAction("xxx"))

就会在进入界面的时候去获取data， 首先我们dispatch了一个LoadingAction, ui会受到通知，并且开始showLoading， 然后我们从repo里面获取model层的模块，这个是一开始建立Store的时候传递进去的context，它可以是网络的retrofit，也可以是数据库的DAO等，数据回来后，我们先用同样方法，取消Loading，然后调用相关数据notifyChange，ui就会收到通知



注意，我们的State必须是data class，所以在这里更新的时候可以调用copy方法去产生一个新的对象，如果State里面字段比较多，只更新某个字段copy方法还是比较方便的。notifyChange实际上是State的一个扩展方法，调用xxState.notifyChange()会让订阅LiveState<xxState>的观察者收到通知



##Middleware

middleware类似okhttp的intercepter， 它是一个action的拦截器，okhttp利用拦截器可以做很多事情，比如请求验证，cookie，mock数据等等，我们同样可以使用middleware做一些操作，比如LifecycleStore里面内置了几个middleware, ActionCheckMiddleware（用于检查action是否合法），

ActionRecordMiddleware（用于在ios crash的时候收集action记录），当然您也能用middleware去做一些更有想象力的事情，自定义的middleware通过构造函数参数传递到store就能被使用

class LogMiddleware<Store : LifecycleStore<*>> : BaseMiddleware<Store>() {

    override fun run(dispatch: DispatchFunction, store: Store, action: IAction) {

        if (ClassChecker.isDataClass(action::class)) {

            ALog.d("MRedux", "$action")

        } else {

            ALog.d("MRedux", "${action::class.simpleName}")

        }

    }

}



上面的例子我们用middleware来收集action信息，并使用ALog存储，如果线上出现问题，可以在Slardar上看到用户发出了哪些action，来追溯用户的行为, 当然，这个middleware已经默认被加入到BaseLifecycleStore中。更多Middleware可以查看前端的文档



##More

一般的来说，使用上面的组件已经可以完成大部分的需求了，事实上我们还提供了以下的组件可供视情况选择使用

SharedAction&reducer

因为Action本身是和Store泛型绑定的，如果我们有Action需要在多个Store之间公用呢？比如某个网络请求需要多处发起。这个时候可以把公共逻辑放到SharedAction，它没有泛型。可能您想在SharedAction中更新ui，可以定义一个纯数据的Action结合Reducer使用

data class UpdateUIAction(val uidata: String): IAction



data class SomeLogicAction: SharedAction{

    fun run(dispatcher: Distributor){

        val result = doSomeThing()

        dispatcher.dispatch(UpdateUIAction(result))

    }

}



class MyReducer : Reducer<XXStore>{

    override fun reduce(store: XXStore, action: IAction) {

        if(action is UpdateUIAction){

            //更新ui

            store.state.xxx

        }

    }

}



val store: XXStore = createLifecycleBindingStore()

store.applyReducer(MyReducer())



##MapReducer

我们的所有LiveState其实是放在一个大的HashMap里面的，所有state会用class+id来唯一标识。所以我们可以在两个id相同的state之间做关联，让一个state变化引起另一个state变化(因为id相同，我们认为他们是有关联的)。比如有以下场景，在列表页面feed流里面有点赞的功能，点击feeditem进入详情页面也能点赞，那么我们希望详情页点赞ui结果能直接反应到feed流上面，即详情页点赞后，列表页自动变成被点赞的状态

class DetailToListMapReducer : MapReducer<DetailState, ListState> {

    override fun reduce(detail: DetailState, oldList: ListState): ListState? {

        //返回一个copy后的新数据即完成详情到列表页更新

        return oldList.copy(field = detail.xx)

    }



    override fun reverseReduce(list: ListState, oldDetail: DetailState): DetailState{

        return oldDetail.copy(field = list.xx)

    }

}



##EventHub

android的EventBus是基于观察值模式的，MRedux也是如此，因此我们的Store其实还有Eventbus

的功能。如果需要可以使用EventHub类

object EventHub{

    fun <E : Event> registerEvent(

        eventClass: KClass<out E>,

        store: LifecycleStore<*>? = null,

        inactiveLost: Boolean = false,

        callback: (E) -> Unit

    ): EventSubscription 



    fun <E : Event> post(event: E)

}

在注册需要观察的event的时候，如果传递的LifecycleStore不是null，那么event事件会和这个LifecycleStore绑定，即store如果destroy了，就不会受到消息了，如果null，则长期观察，需要手动解除注册，否则会内存泄露。用EventHub会让Event在所有Store上传递，如果您想让一个Event只在一个特定的Store上传递，那么可以使用store.register和store.post



##End

我们还是不得不承认框架上还是有一些不足之处的，比如对于ui来说，监听数据可以会比较冗长，比如store.xxLiveState.state.copy().notifyChange，再比如相比mvp来说我们引入了更多的概念，

有一定学习成本。提升开发者的体验，我们也在持续优化中...
		