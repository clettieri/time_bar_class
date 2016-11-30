# time_bar_class
A class for managing real time price ticks into a OHLC bar based on time.  Converts real time ticks into price bars.

This class is designed to be used in conjunction with a main script that receives
a stream of tick data in real time.  Please read comments thouroughly as there
are a few things you must implement in your main script for this to work.

HOW TO USE
-Assumes time_bars is a dictionary of {'Symbol' : TimeBar objects}
-Assumes sym is the symbol.ticker string of the stock

In main script:
    -Initialize TimeBar object for each symbol
            On Symbol Update/ Tick Received
                -Call new_print to append the print object
                    -time_bars[sym].new_print(symbol.last_print)
	-Create bar_timer() in main script (see below)
		This function is what calls the bar_closed()
	-Start the bar_timer thread

                    
USING THE BARS
In main script
    -bar_list = time_bar[ticker].get_bar_list()
    -bar_list[-1] = most recent closed bar
    -bar_list[-1]['high'] = high of most recent closed bar
    -bar attributes are 'open', 'high', 'low', 'close', 'close_time'
    -bar close time is the real local time that a new print came in and told
    this class to close the bar (round off seconds if needed)
	
LIMITATIONS
*Need seperate TimeBar Object for each symbol
*Bar interval can be minutes only for now (1-30)
###########################

** bar_timer function **
This is the thread function that should handle all timing responsibilties and run in main script:

REQUIRES global variables:
BAR_INTERVAL = int
TIME_BAR_DICT = dictionary if string : TimeBar object
optional - is python logger setup (remove statements with logging if you dont want)

def bar_timer():
    ''''''
	Called in seperate thread, called after initializes all the bars.
    Will call bar_close() in every time bar.  This function requires a global
    BAR_INTERVAL which is the n mintues of the time bars, and the TIME_BAR_DICT
    ''''''
    def set_first_bar_close():
        ''''''run on intialization, uses system time and rounds off seconds to 
        set global next bar close time.  This was used mainly in older time
        bar classes - not needed with the thread timer''''''
        now = datetime.now() + timedelta(minutes=1)
        while now.minute % BAR_INTERVAL != 0:
            #increment by a minute until its proper interval
            now += timedelta(minutes=1)
        return now.replace(second=0, microsecond=0)
        
    def increment_next_bar_close_time(previous_time):
        ''''''will increment the property next bar close time by the bar interval''''''
        previous_time += timedelta(minutes=BAR_INTERVAL)
        return previous_time
    logging.info("Bar time thread starting..")
    #Sleep until start of first bar - so start with full bar
    seconds_to_sleep = 0
    now = datetime.now()
    target_time = set_first_bar_close() #First bar close
    logging.info("Initial target time: " + str(target_time))
    delta = target_time - now
    seconds_to_sleep = delta.total_seconds()
    sleep(seconds_to_sleep)
    #TODO - WARNIONG - the first bar will ahve ticks from the previous fraction bar here
    #TODO - WARNING - this is where I can loop through and clear every symbols tick_list before starting the full bar
    #TODO- WARNING - but since it will be premarket I dont think it is sucha big deal to have fraction of bar here
    #Now loop through every full interval and build bars
    while True:
        target_time = increment_next_bar_close_time(target_time)
        logging.info("Next target time: " + str(target_time))
        now = datetime.now()
        delta = target_time - now
        sleep(delta.total_seconds())
        #time it here to end of for loop, how long to close all bars?
        for ticker in TIME_BAR_DICT:
            TIME_BAR_DICT[ticker].bar_closed()

bar_timer_thread = Thread(target=bar_timer)
bar_timer_thread.start()

