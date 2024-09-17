# UEFN-Custom-Timer
UEFN verse Custom Countdown Timer - with callback functions

I've created a custom timer class to replace the timer device in the game.

I've added a lot of functionality to it. We can register events in this timer for a callback every second, and a callback at the end of the timer (similar to the broadcast-listener pattern).

We can also have the Timer displayed on the UI. Moreover, I've also added an urgency mode (the text will turn red).

You asked me why I rewrote the timer device？ because the original timer device in the game CANNOT BE MODIFIED!

For example, I've added a custom UI to the code, so that we can change its position, add backgrounds, etc.

Even if we don't need that functionality, we can still register a secondly timer event in another location to perform the function we want.

And it doesn't need to be placed in the game, it just needs to be referenced in the code, making it neater.

Here is the full code snippet:
=
***********************************************************************************************
        
    using { /Fortnite.com/Devices }
    using { /Verse.org/Simulation }
    using { /UnrealEngine.com/Temporary/Diagnostics }
    using { /Verse.org/Verse }
    using { /UnrealEngine.com/Temporary/UI }
    using { /Fortnite.com/UI }
    using { /Verse.org/Colors/NamedColors }
    using { /UnrealEngine.com/Temporary/SpatialMath }
    
    
    # Function called on every secound, with param of actual secounds
    sec_fun := type{_(:int):void}
    
    #Function called on countdown finish
    end_fun:= type{_():void}
    
    #custom count down timer
    countdown_timer := class():
    
        #List functions subscribe
        var SecFuns<private>:[]sec_fun = array{}
        var EndFuns<private>:[]end_fun = array{}
        
        #Timer vars
        var TimeCount :int = 0
        var TimerMin:int = 0 
        var TimerSec:int = 0 
        var BreakLoop :logic = false
        var IsSuccess:logic = false
        
        #Timer UI
        var MaybeUIPerPlayer : [player]?canvas = map{}    #map for player with their UI
        var UICanvas: canvas = canvas{}
        var TextBlockTimer : text_block = text_block{DefaultTextColor := Beige}     #UI Widgets
        var IsShowUI:logic = false
        var MyCustomText:string  = ""
        var UrgencyModTime:int = 0
    
    #Public functions:
    
        #Subscribe function called on each secound
        SubscribeSecFunction<public>(Fun:sec_fun):void =
            set SecFuns+= array{Fun}
    
        #Subscribe function called on countdown end
        SubscribeEndFunction<public>(Fun:end_fun):void =
            set EndFuns+= array{Fun}
    
        #Start timer with the time given and show timer on screen of agents given. CostomText: Add a custom text before time. UrgencyTime: Text color will become in red with in urgency time
        StartTimer<public>(SecoundsCount:int,?CanShowUI:logic = false,?AgentList:[]agent = array{},?CustomText:string = "", ?UrgencyTime:int = 0):void = 
            set TimeCount = SecoundsCount
            set IsShowUI = CanShowUI
            set MyCustomText = CustomText
            set UrgencyModTime = UrgencyTime
    
            if(IsShowUI?):
                for(Agent:AgentList):
                    CreateUIPlayer(Agent)
    
            spawn:
                OnCountDownTimer()
    
        StopTimer<public>(): void = 
            set BreakLoop = true
    
    #Private functions:
    
        OnCountDownTimer<private>()<suspends>:void = 
            set BreakLoop = false
            set IsSuccess = false
            loop:
                Sleep(1.0)
                if(TimeCount>1):
                    set TimeCount -=1
                    DispatchSecFunction()
                    if(IsShowUI?):
                        UpdateTimerUI()
                else:
                    set BreakLoop = true
                    set IsSuccess = true
    
                #Call time complete functions
                if(BreakLoop?):
                    if(IsSuccess?):
                        DispatchEndFunction()
                    ResetTimer()
                    break
                    
        DispatchSecFunction<private>():void =
            for (Fun:SecFuns): 
                Fun(TimeCount) #Time reste (sec)
    
        DispatchEndFunction<private>():void =
            for (Fun:EndFuns): 
                Fun()
    
        # A custom UI can only be associated with a specific player, and only that player can see it
        CreateUIPlayer<private>(Agent : agent) : void =
            if (Player := player[Agent]):
                if(not(PlayerExists := MaybeUIPerPlayer[Agent]?)): #if player haven't an UI canvas exist
                    if(PlayerUI := GetPlayerUI[Player]):    
                        MyCanvas := canvas:
                            Slots := array:
                                #Timer text slot
                                canvas_slot:
                                    Anchors := anchors{Minimum := vector2{X := 0.5, Y := 0.0}, Maximum := vector2{X := 0.5, Y := 0.0}}
                                    Offsets := margin{Top := 145.0, Left := 0.0, Right := 0.0, Bottom := 0.0}
                                    Alignment := vector2{X := 0.5, Y := 0.0}
                                    SizeToContent := true
                                    Widget := TextBlockTimer
    
                        Print("Ui should be added")
                        set UICanvas = MyCanvas
                        PlayerUI.AddWidget(UICanvas)
                        if (set MaybeUIPerPlayer[Player] = option{UICanvas}):
                            TextBlockTimer.SetTextColor(Beige)
                            UpdateTimerUI()
    
        #Call on each secound
        UpdateTimerUI<private>():void = 
            if(ifMin:int = Floor(TimeCount/60)):
                set TimerMin = ifMin
            if(ifSec:int = Mod[TimeCount,60]):
                set TimerSec = ifSec 
            var TimeTxt:string = ""
            if(TimerSec>=10):
                set TimeTxt = "{TimerMin}:{TimerSec}"
            else:
                set TimeTxt = "{TimerMin}:0{TimerSec}"
    
            TextBlockTimer.SetText(StringToMessage(MyCustomText+TimeTxt))
            if(TimeCount<=UrgencyModTime):
                TextBlockTimer.SetTextColor(Red)
    
        #Call on the end of timer
        ResetTimer():void =
            #Clear all tasks
            set SecFuns = array{} 
            set EndFuns = array{}
            #Remove player's UI
            if(IsShowUI?):
                for(Key->Value:MaybeUIPerPlayer):
                    if( PlayerUI := GetPlayerUI[Key]):
                        PlayerUI.RemoveWidget(UICanvas)
                        if(set MaybeUIPerPlayer[Key] = false){}
            


So, how to use this timer? Very easy!

Here I've wrote a test device, testing with a trigger_device and hud_message_device:

I start by instantiating a custom timer and touching the trigger to turn the timer on.

Before the timer launches I register a time-per-second event and an end-of-timer event for it (because each time the timer clears the event list after it ends, you need to register the events each time before using the timer)

When launching the timer we need to provide the duration of the timer (in seconds) We can select to not display the timer in the UI (commented code below) or we can select to use the timer UI, we need to provide a list of Agents that need to be displayed, custom string text (to display in front of the timer), and whether or not to enable the urgent time.

The format of the functions that need to be registered is provided above the countdown_timer class, where the current events per second need to be registered as functions with a parameter of int, in which the parameter is the remaining time. The end-of-time event needs to be registered as a function with null parameters and return value.

The types of these functions can be modified as needed.

Here is the test code (use case):
=
************************************
    using { /Fortnite.com/Devices }
    using { /Verse.org/Simulation }
    using { /UnrealEngine.com/Temporary/Diagnostics }
    using { /Fortnite.com/Playspaces }
    
    delegate_void:= type{_():void}
    
    test_timer_device := class(creative_device):
    
        
        Trigger: trigger_device = trigger_device{}
        
        HudMessage: hud_message_device = hud_message_device{}
        
        TimerDevice:countdown_timer = countdown_timer{}
    
        OnBegin<override>()<suspends>:void=
            Trigger.TriggeredEvent.Subscribe(OnTriggered)
    
        #Test custom timer event with UI
        OnTriggered(Agent:?agent):void=
            TimerDevice.SubscribeSecFunction(UpdateEverySec)
            TimerDevice.SubscribeEndFunction(OnTimeFinish)
    
            if(MyAgent:=Agent?):
                TimerDevice.StartTimer(70,?CanShowUI:=true,?AgentList:=array{MyAgent},?CustomText:="Time count down:",?UrgencyTime:=10)
            
        UpdateEverySec(TimeLeft:int):void = 
            HudMessage.SetText(StringToMessage("Time Left: {TimeLeft} s"))
            if(TimeLeft<=3):
                TimerDevice.StopTimer()
    
        OnTimeFinish():void = 
            HudMessage.SetText(StringToMessage("Do sth on time end!!!"))
    
    <#
        Test custom timer event without UI
        OnTriggered(Agent:?agent):void=
            TimerDevice.SubscribeSecFunction(UpdateEverySec)
            TimerDevice.SubscribeEndFunction(OnTimeFinish)
            TimerDevice.StartTimer(80)
            
        UpdateEverySec(TimeLeft:int):void = 
            HudMessage.SetText(StringToMessage("Time Left: {TimeLeft} s"))
    
        OnTimeFinish():void = 
            HudMessage.SetText(StringToMessage("Do sth on time end!!!"))
            #>

In the test we can observe that after touching the trigger the Timer starts running. the Timer's UI is displayed on the top and at the same time the Hud message device starts triggering the display events per second. The text turns red with 10 seconds left. The end event is triggered when the timer ends.

![image](https://github.com/user-attachments/assets/5352c52e-ccb0-4a58-b603-00a9a29399e9)

Custom UI with custom function per sec

![image](https://github.com/user-attachments/assets/64f6ae5a-7e90-4b7b-b398-609265d41c28)

Custom UI Urgency mod

## IMPORTANT - Do not subscribe& launch the same timer in a subscribed time-end-event!!!!
We can't launch a new timer on the end event of a registered timer. in fact, I tried to create a timer loop (at the end of a timer, register and launch a new timer). But don't forget that the events registered in a timer will be cleared when the events are all executed, i.e. it's not possible to register new functions in the end event of the same timer.

The solution is:

- When registering a new timer in the time-end-event, call a suspends method with a 0.1 second delay. This will ensure that the timer is cleared before it is registered.

- Or we can create two timer instances and call each other for a loop.

