# Readme
to understand how logsummary parse the debug file and generate HTML or CVS file, we have to first understand [Finite State Machine](#fsm), which we use as a framework for this parsing task, and then we are able to [write the log file](#lsw) depending on the states of Finite State Machine. Following, I will show two examples of [how to get more information from debug files](#eg).

## [Finite State Machine](id:fsm)
The Finite State Machine we defined is a class initialized with a starting state: `fsm = FiniteStateMachine(state_normal)`, and each state has several possible next states to transit to, as well as corresponding functions to be called, which [I will describe in detail later](#howto).

	if 'on_match_go_to_state' in match:
    	self.state = match['on_match_go_to_state']

    if 'on_match_functions' in match:
    	for function in match['on_match_functions']:
        	function(line_data)

In where, **match** is a **LineMatch** object that explains how to take an arbitrary string and if it matches a regular expression, *pull out specific data from the regular expression* and *put it into a dictionary*. The concept of **match** is the key of understanding how this program works, and it might be a little confused at first glance, so here's an example:

We design a regular expression

	re_api_call_start = re.compile(
      '^(.{26}).+ S[Yy][Mm][\w]{0,5} (Sym[A-Za-z0-9]+|taskRun|taskSetup|emc[A-Za-z]+)\s+([Ff]unction [Ee]nter|@\(#\)|[Ea-z\s]+ \(task NDM.*\))(.*)')
     
and a **LineMatch** object

	line_match_api_call_start = LineMatch
		(re_api_call_start, {'date': 1, 'api_call': [None, 	2, 4],'contents': 'api_call_start'})

which means that if we run into the following line in a debug file

	10/21/2015 14:26:52.210814   3937      1           EMC Corp                     SYMDM taskSetup                         Entered function (task NDM:DARYL_       NDM_SG) SIDs 000194901137 n/p, flags TRANSIENT,QUERY_ONLY
	
`re_api_call_start` would capture 4 matched groups, which are

1. 10/21/2015 14:26:52.210814  
2. taskSetup  
3. Entered function (task NDM:DARYL_ NDM_SG)  
4. SIDs 000194901137 n/p, flags TRANSIENT,QUERY_ONLY

and the *specific data we want to put into a dictionary* is indicated by the **LineMatch** object **line_match_api_call_start**, which is the **line_data**

| Key          | Value                     |
|:------------:|:-------------:            |
| 'date'       | 10/21/2015 14:26:52.210814|
| 'api_call'   | taskSetup SIDs 000194901137 n/p, flags TRANSIENT,QUERY_ONLY |
|'contents'    | 'api_call_start'          |

In the future, we will write all the **line_data** into log file, as [we can see later](#write_line).
### [Add state transition and corresponding functions](id:howto)
Since we have initialized the Finite State Machine with the `state_normal`, you must be curious about how we transit from state to state, we do that through the `add_transition` method of Class **State** as follows

	state_api_call_start.add_transition(
      {'line_match': line_match_api_call_start, 'on_match_go_to_state': state_api_call_start,
       'on_match_functions': [LogSummary.set_line_number, FiniteStateMachine.write_line]})
       
which simply means that if we are at state `state_api_call_start`, and the next string matches with **match['line_match']** (i.e. line_match_api_call_start), the Finite State Machine would transite to **match['on_match_go_to_state']** (i.e. state_api_call_start), and then run all of the functions in **match['on_match_functions']** (i.e. first LogSummary.set_line_number(), then FiniteStateMachine.write_line()). By far, the Finite State Machine has been set up.

## [Log Summary Writer](id:lsw)
Once we make the Finite State Machine running as expected, we can generate the summary log files using Class **LogSummaryWriter**.  
Still use the example above, because **line_data['contents']** is 'api_call_start'

	if line_data['contents'] == 'api_call_start':
      if LogSummary.line_cache == []:
        LogSummary.line_cache.append(copy.deepcopy(line_data))

      else:
        LogSummary.line_cache.reverse()
        for line_data_tmp in LogSummary.line_cache:
          sys.stdout.write('"%s","%s Starts %s","","","","","","%d"\n' % (
            line_data_tmp['date'], line_data_tmp['api_call'], line_data_tmp['api_call_info'],
            line_data_tmp['line_no']))
        LogSummary.line_cache = [copy.deepcopy(line_data)]

      return
      
This code of Class **LogSummaryWriter** simply means that if there is other **line_data** in **LogSummary.line_cache**, first write them in the log file reversly, and then put the holding **line_data** into **LogSummary.line_cache**. This is how we write the captured information of one state into the log file, and [I will describe how we write additional information later](#eg), as an example in case that the readers want to add other useful information later.

### [Write Additional Information Examples](id:eg)
As an illustration, I will demonstrate how I extract [taskTerminate](#task_terminate) api call information first, and then the [task NDM](#task_ndm).

#### [taskTerminate](id:task_terminate)  
First, we would like to design a regular expression that can capture the taskTerminate from the debug file

	re_task_terminate = re.compile('^(.{26}).+ (taskTerminate)\s+(.*)')
	
which would give us three matched groups when it run into the corresponding line in debug file

1. 10/21/2015 14:26:56.210392
2. taskTerminate
3. Task has terminated (task NDM:DAR YL_NDM_SG)

And then design the **LineMatch** object

	line_match_task_terminate = LineMatch(re_task_terminate,
                                          {'date': 1, 'api_call': 3, 'contents': 'task_terminate'})
                                          
which would give us the **line_data** as follows

| Key          | Value                     |
|:------------:|:-------------:            |
| 'date'       | 10/21/2015 14:26:56.210392|
| 'api_call'   | Task has terminated (task NDM:DAR YL_NDM_SG) |
|'contents'    | 'task_terminate'          |

And then the last step would be involving the state into the Finite State Machine according to some transition rules, in this case, it would stand still at state `state_normal`, so no need to create a new state. However, we still need the method `add_transition` to tell us what functions need to be called in this case.

	state_normal.add_transition(
      {'line_match': line_match_task_terminate, 'on_match_go_to_state': state_normal,
       'on_match_functions': [FiniteStateMachine.write_line]})
       
From where, we can tell that if the Finite State Machine is at state `state_normal`, and the next string matches with **line_match_task_terminate**, it will stay at `state_normal`, and write the corresponding **line_data** into log file.

####[task NDM](id:task_ndm)
Unlike the last example, since the task NDM and its corresponding nugget calls are also api call, we would like to design new regular expression, but modify the existing regular expression of api call would be more considerable.

	re_api_call_start = re.compile(
      '^(.{26}).+ S[Yy][Mm][\w]{0,5} (Sym[A-Za-z0-9]+|taskRun|taskSetup|emc[A-Za-z]+)\s+([Ff]unction [Ee]nter|@\(#\)|[Ea-z\s]+ \(task NDM.*\))(.*)')
      
And the **LineMatch** object

	line_match_api_call_start = LineMatch(re_api_call_start,
                                          {'date': 1, 'api_call': [None, 2, 4], 'contents': 'api_call_start'})
                                          
which would give us the **line_data** as follows

| Key          | Value                     |
|:------------:|:-------------:            |
| 'date'       | 10/21/2015 14:26:52.210814|
| 'api_call'   | taskSetup SIDs 000194901137 n/p, flags TRANSIENT,QUERY_ONLY |
|'contents'    | 'api_call_start'          |

Involving this into the Finite State Machine would be the same as [mentioned before](#howto).  

The last step before we are done with this task is to modify the method [**write_line**](id:write_line) of class **LogSummaryWriter**. It because that we want to get the information when entering a function (action, flags etc.), so we need to get them from **LogSummary.line_cache** when they exists.

	temp = LogSummary.line_cache[-1]
    temp = temp['api_call'].split(' ')
    temp.pop(0)

    sys.stdout.write('<tr><td>%s</td><td colspan="6">%s&nbsp%s&nbsp%s</td><td>%d</td></tr>\n' % (
    				  line_data['date'], line_data['api_call'], "[Entering with "+' '.join(temp)+"]", " Starts/Ends", line_data['line_no']))
    LogSummary.line_cache.pop()
    
in where, the `temp` in first line is the **line_data** containing the information of entering a function.

## Extention for Python 3
In case that there is someone who want to use python 3, here is the solutions.

First we use the shebang notation, which can release us from typing the interpretor name every time we want to run the script, which is as follows:

	#!/usr/bin/env/python3
	
the path after `#!` is the location of python 3 interpretor.

Further more, due to the PEP 0008, Python 3 disallows mixing the use of tabs and spaces for indentation, so please use spaces as the indentation method when extending this script. 

## Conclusion
It is a very smart way of warpping the whole parsing procedure as a Finite State Machine, even though it is a little bit convoluted and not easy for people to read, for which reason, that I hope this readme file would be helpful of getting people understand and extent it in the future. And of course, please let me know if there is anything wrong in the file, just email me: <cheng.gu@emc.com>