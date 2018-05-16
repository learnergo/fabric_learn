### FSM

FSM是finite state machine的缩写，有限状态机，是ChaincodeSupport服务使用到的一个第三方库，在github.com/looplab/fsm可以下载。FSM将一个事物从状态A向状态B的转化看作一个事件，并可以设置在进入/离开某个状态时自动调用的时机函数。每个状态事件、状态、时机函数都用字符串关键字表示。在此简单介绍一下用法：


```
//创建一个状态机
//三个参数：1.默认状态 2.定义状态事件 3.定义状态转变时调用的函数
fsm := fsm.NewFSM(
    "green",
    fsm.Events{
        //状态事件的名称   该事件的起始状态Src         该事件的结束状态Dst
        //即：状态事件warn（警告事件）表示事物的状态从状态green到状态yellow
        {Name: "warn",  Src: []string{"green"},  Dst: "yellow"},
        {Name: "panic", Src: []string{"yellow"}, Dst: "red"},
        {Name: "calm",  Src: []string{"red"},    Dst: "yellow"},
    },
    //状态事件调用函数，在此称为 时机函数。关键字用'_'隔开，格式是："调用时机_事件或状态"
    //before表示在该事件或状态发生之前调用该函数，如"before_warn"，表示在warn
    //这个状态事件发生前调用这个函数。"before_yellow"表示进入yellow状态之前调用
    //该函数。
    //依此类推，after表示在...之后，enter表示在进入...之时，leave表示在离开...
    //之时。
    fsm.Callbacks{
        //fsm内定义的状态事件函数，关键字指定的是XXX_event和XXX_state
        //表示任一的状态或状态事件
        "before_event": func(e *fsm.Event) {
        fmt.Println("before_event")
        },
        "leave_state": func(e *fsm.Event) {
        fmt.Println("leave_state")
        },
        //根据自定义状态或事件所定义的状态事件函数
        "before_yellow": func(e *fsm.Event) {
        fmt.Println("before_yellow")
        },
        "before_warn": func(e *fsm.Event) {
        fmt.Println("before_warn")
        },
    },
)
//打印当前状态，输出是默认状态green
fmt.Println(fsm.Current())
//触发warn状态事件，状态将会从green转变到yellow
//同时触发"before_warn"、"before_yellow"、"before_event"、"leave_state"函数
fsm.Event("warn")
//打印当前状态，输出状态是yellow
fmt.Println(fsm.Current())
```
