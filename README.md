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
=================================================================================================================================
    
    using { /Fortnite.com/Devices }
    using { /Verse.org/Simulation }
    using { /UnrealEngine.com/Temporary/Diagnostics }
    using { /Verse.org/Verse }
    using { /UnrealEngine.com/Temporary/UI }
    using { /Fortnite.com/UI }
    using { /Verse.org/Colors/NamedColors }
    using { /UnrealEngine.com/Temporary/SpatialMath }
    
    
    #Function called on every secound, with param of actual secounds
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
            loop:
                Sleep(1.0)
                if(TimeCount>1):
                    set TimeCount -=1
                    DispatchSecFunction()
                    if(IsShowUI?):
                        UpdateTimerUI()
                else:
                    set BreakLoop = true
    
                #Call time complete functions
                if(BreakLoop?):
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
            
