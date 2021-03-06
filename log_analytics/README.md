## Scripts to extract and analyze Watson Assistant logs
These scripts are intended to form a data pipeline

# getAllLogs.py
Using a filter argument grabs a set of logs from Watson Assistant.  Specify the maximum number of log pages (`-n`) to retrieve and the maximum number of log entries per page (`-p`).

This utility uses the `list_logs` API when a workspace_id is passed, otherwise uses `list_all_logs` API which requires a language and one of `request.context.system.assistant_id`, `workspace_id`, or `request.context.metadata.deployment` in the filter (see https://cloud.ibm.com/apidocs/assistant/assistant-v1?code=python#list-log-events-in-all-workspaces)

The `filter` syntax is documented here: https://cloud.ibm.com/docs/services/assistant?topic=assistant-filter-reference#filter-reference

Example for one workspace:
```
python3 getAllLogs.py -a your_api_key -w your_workspace_id -l https://gateway-wdc.watsonplatform.net/assistant/api -c raw -n 100 -p 100 -o 10000_logs.json -f "response_timestamp>=2019-11-01,response_timestamp<2019-11-21"
```

Example for one assistant:
```
python3 getAllLogs.py -a your_api_key -l https://gateway-wdc.watsonplatform.net/assistant/api -c raw -n 100 -p 100 -o 10000_logs.json -f "language::en,response_timestamp>=2019-11-01,response_timestamp<2019-11-21,request.context.system.assistant_id::your_assistant_id"
```

The Watson Assistant team has put out a similar script at https://github.com/watson-developer-cloud/community/blob/master/watson-assistant/export_logs.py

# extractConversations.py
Takes a series of logs, groups them by a conversation unique identifier, and outputs a new JSON file.  The new JSON file has keys for the unique conversation ids, and the value associated to each key is a list of log events for that conversation id.

This will remove Watson Assistant logs that do not belong to a conversation as indicated by the "primary key" parameter.

The input parameter can be a single JSON file containing log events or a directory containing multiple JSON files.

The unique conversation identifier is provided with `-c`.  Note that if a single conversation spans multiple workspaces (skills), you cannot use `conversation_id` as the unique identifier.

Example for text-based assistants:
```
python3 extractConversations.py -i 10000_logs.json -o 10000_logs_by_conversation_id.json -c "response.context.conversation_id"
```

Example for voice-based assistants using IBM Voice Gateway:
```
python3 extractConversations.py -i 10000_logs.json -o 10000_logs_by_conversation_id.json -c "request.context.vgwSessionID"
```

# logIntentStats.py
Quick and dirty summarization of log data, reads a file produced by `extractConversations.py` (with conversation IDs as keys and list of log events as values and builds multiple reports) and produces some summary statistics.

Requires knowing which turn index of the conversation is the first message from the user (the turn index is 0-based), this index is specified as the "first turn index" variable (`-f`).  For most assistants this value is `1`.
* If the assistant does not say anything until the user sends text, the value is `0`.
* If the conversation starts with a system greeting (Ex: "How can I help you") from the assistant, the value is `1`.
* If an orchestration layer sends text to the assistant before the user does, the value may be `2` or higher.

Optionally specify `--speech_confidence_variable` to indicate which context variable contains the speech transcription confidence (A common pattern is to use `request.context.STT_CONFIDENCE`).  If this parameter is omitted the log analysis will not include any speech metrics.

The reports are as follows:
* `raw-intent-turn.tsv`: First turn utterance, intent, confidence. Helps find new training/test data for the classifier.
* `first-turn-stats.tsv`: Intent summary gathered from first conversation node only, summarizing totals, classifier confidence, and STT confidence.
* `all-intent-turn.tsv`: Utterance, intent, confidence for every conversational turn.

Example invocation:
```
python3 logIntentStats.py -i 10000_logs_by_conversation_id.json -f 1
```

# intent_heatmap.py
Takes a tab-separated file (ie from `logIntentStats.py`) and builds heat maps to help visualize the intent metrics.

```
python3 intent_heatmap.py -i first-turn-stats.tsv -o intent_conf.png -s "Total" -r "Intent Confidence" -l "Intent" -t "Classifier confidence by intent"
python3 intent_heatmap.py -i first-turn-stats.tsv -o stt_conf.png -s "Total" -r "STT Confidence" -l "Intent" -t "Speech confidence by intent"
```

# printLog.py
Takes a log file grouped by conversations and prints a quick turn-by-turn summary of that conversation (the text passed between user and system) based on a conversation id.

Example:
```
python3 printLog.py 10000_logs_by_conversation_id.json 12345678-4567-7890-abcdefghijklmno
```


# Other analyses
Several other types of analysis are possible with Watson Assistant log data.  The Watson Assistant development team has released [two notebooks](https://github.com/watson-developer-cloud/assistant-improve-recommendations-notebook) which help further analyze log data:
* Measure Notebook - The Measure notebook contains a set of automated metrics that help you monitor and understand the behavior of your system. The goal is to understand where your assistant is doing well vs where it isn’t, and to focus your improvement effort to one of the problem areas identified.
* Effectiveness Notebook - The Effectiveness notebook helps you understand relative performance of each intent and entity as well as the confusion between your intents. This information helps you prioritize your improvement effort.
* The [Dialog Skill Analysis](https://medium.com/ibm-watson/announcing-dialog-skill-analysis-for-watson-assistant-83cdfb968178) helps assess your Watson Assistant intent training data for patterns before you deploy.
* [Analyze chatbot classifier performance from logs](https://medium.com/ibm-watson/analyze-chatbot-classifier-performance-from-logs-e9cf2c7ca8fd) is a blog and video demonstrating how you can take utterances from logs and convert them into training and/or testing data.
* Beware of focusing too much on training accuracy - you should measure and be concerned about blind test accuracy.  See this two-part blog series on "Why Overfitting is a Bad Idea and How to Avoid It" with [Part 1: Overfitting in General](https://medium.com/ibm-watson/why-overfitting-is-a-bad-idea-and-how-to-avoid-it-part-1-overfitting-in-general-b8a3f9ffcf66) and [Part 2: Overfitting in Virtual Assistants](https://medium.com/ibm-watson/why-overfitting-is-a-bad-idea-and-how-to-avoid-it-part-2-overfitting-in-virtual-assistants-a30f4d999adc).

