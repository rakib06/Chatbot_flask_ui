# rasa-chatbot 
This is a short tutorial to show how I create a chatbot on my local server using Rasa NLU, Rasa Core, FLASK and ngrok. It will cover setting up rasa, setting up webchat, brief intro to rasa, using custom actions and use ngrok to deploy this dev server temporarily. 

## Set up
### Rasa Set up
```
virtualenv venv36 -p "C:\Program Files\Python36\python.exe" 
# 64 bit python 

mkdir rasa
cd rasa
```
create requirements.txt and paste this inside
```
rasa_core
rasa_sdk
rasa_core_sdk
rasa_nlu[spacy]
nltk
numpy
pandas
scikit-learn
flask
gunicorn
```
then run `pip install -r requirements.txt` else you can do it through `pip install`. Then run `rasa init --no-prompt` to set up a default rasa. 

Create a server.py file in rasa and copy this inside.
```
from rasa_core.channels import SocketIOInput
from rasa_core.agent import Agent
from rasa_core.interpreter import RegexInterpreter
from rasa_core.interpreter import RasaNLUInterpreter
from rasa_core.utils import EndpointConfig
from rasa_core.policies import KerasPolicy, MemoizationPolicy

# load your trained agent
interpreter = RasaNLUInterpreter('rasa/models/nlu/profiler/nlu')
agent = Agent.load('rasa/models/dialogue', 
	interpreter=interpreter, 
	action_endpoint=EndpointConfig('http://localhost:5055/webhook')
    #for action endpoint
)

input_channel = SocketIOInput(
    # event name for messages sent from the user
    user_message_evt="user_uttered",
    # event name for messages sent from the bot
    bot_message_evt="bot_uttered",
    # socket.io namespace to use for the messages
    namespace=None
)

# set serve_forever=True if you want to keep the server running
s = agent.handle_channels([input_channel], 5500, serve_forever=True)
```

### Webchat set up
Implemented with [react-chat-widget](https://github.com/mrbot-ai/rasa-webchat)
1. Create a Flask App

    Create a file called app.py in the main directory
    ```
    from flask import Flask  
    from flask import render_template

    # creates a Flask application with name "app"
    app = Flask(__name__, static_url_path='/static')

    # a route to display our html page called "index.html" gotten from [react-chat-widget](https://github.com/mrbot-ai/rasa-webchat)
    @app.route("/")
    def index():  
        return render_template('index.html')

    # run the application
    if __name__ == "__main__":  
        app.run(debug=True)
    ```

2. Create a folder called `templates`

    Since Flask app looks for HTML files from a folder named templates by default.

3. Create a HTML file inside templates folder called `index.html`
4. Paste this html file inside which is modified from the template: [react-chat-widget](https://github.com/mrbot-ai/rasa-webchat)
    ```
    <!doctype html>
    <html>
    <head>
    </head>
    <body>
        <div id="webchat">
            <script src="https://storage.googleapis.com/mrbot-cdn/webchat-0.5.8.js"></script>
            <script>
            WebChat.default.init({
                selector: "#webchat",
                initPayload: "/get_started",
                interval: 1000, // 1000 ms between each messageas this will be added to the socket
                socketUrl: "http://localhost:5500",
                socketPath: "/socket.io/",
                title: "Title",
                subtitle: "Subtitle",
                inputTextFieldHint: "Type a message...",
                connectingText: "Waiting for server...",
                hideWhenNotConnected: true,
                fullScreenMode: false,
                profileAvatar: "http://to.avat.ar",
                openLauncherImage: 'myCustomOpenImage.png',
                closeLauncherImage: 'myCustomCloseImage.png',
                params: {
                images: {
                    dims: {
                    width: 300,
                    height: 200,
                    }
                },
                storage: "local"
                }
            })
            </script>
        </div>
    </body>
    </html>
    ```
If you want to have a different profile picture of the bot, you can change "profileAvatar:" link to a different link or link it to a bot from a local storage. In my case, I have done this: `profileAvatar: "/static/bot.png",` where I saved a photo under static folder called bot.png.'

Now you should have folder looking like this
```
rasa-chatbot (my dir name)
    rasa (autogenerated from the cmd earlier)
    templates
        index.html
    static
        bot.png
    app.py
    requirements.txt
```

## About Rasa

There are many files generated after initialising Rasa. We will only be using a few of them.
1. data/nlu.md

    This is the folder where you give examples of how the specific intents can be asked. For Intent (think of this as your intention to do something) like "greet", you would be asking "hi", "hello", "nice to meet you" and so on. This is where you list out these examples.
    ```
    ## intent:greet
    - hey
    - hello
    - hi
    ```
2. data/stories.md

This is where you are basically narrating. It is where you give examples of how the flow of the chat can go. This is where you write down as many possible path of the chatflow as you can. 
```
## happy path
* greet
  - utter_greet
* mood_great
  - utter_happy
```
3. domain.yml

You can basically think of this as the library. This is where you declare everything. All the intents that you are planning to use in your stories and nlu list them under intents. All the `actions` refer to what the bot will do when it detects a certain intent. Templates are for special kind of actions called utterances which is what the bot will reply. This is where utterances that were under `actions` get defined. 

There are also slots and entities that can be defined in this file. 
```
intents:
  - greet
  - goodbye


actions:
- utter_greet
- utter_cheer_up


templates:
    utter_greet:
    - text: "Hey! How are you?"

    utter_cheer_up:
    - text: "Here is something to cheer you up:"
        image: "https://i.imgur.com/nGF1K8f.jpg"
```
4. endpoints.yml

    In this tutorial, we will only need to set one endpoint for custom action server.

5. actions.py

    This is where we can define all our custom actions to send to users when they ask something. A default template is generated.
6. config.yml

    Here is where we define the language, pipeline and the policies we are going to be using. In this tutorial, we are going to use `spacy_sklearn` instead of `supervised_embeddings`. So please replace as such:
    ```
    pipeline: spacy_sklearn
    ```
    This is where we can also define policies like fallback policy or twostage fallback policy for better intent handling. 

## Define initial payload for webchat
Let's create one additional intent called /get_started which is going to appear at the start of the chat. (Since the index.html's initial payload is called get_started)
1. Include an intent called `get_started` in the `domain.yml` file. Also append an utterance called `utter_intro` under the section called action so that the bot can introduce itself in the beginning. Then define this utterance in the template section of the same file.
    ```
    intents:
    - greet
    - get_started
    - goodbye
    - affirm
    - deny
    - mood_great
    - mood_unhappy

    actions:
    - utter_intro
    - utter_greet

    templates:
    utter_intro:
    - text: "Hi! My name is rsykoss :) Nice to meet you!"
    
    ```
2. Include this intent with the utterance at the start of every story in `stories.md` so that it will appear at the start. For example, the first path will look like this. Do this for all the other stories.
    ```
    ## happy path
    * get_started
    - utter_intro
    * greet
    - utter_greet
    * mood_great
    - utter_happy

    ## sad path 1
    * get_started
    - utter_intro
    .........
    ```
    Now we have the initial payload so once we load the webchat, the bot will say "Hi! My name is rsykoss :) Nice to meet you!" first before you type in anything else. 

## Rasa Train & Test
### Using terminal
Now that we have the default template, we can test it out. 
First let us use the cmd to train then test. 
```
cd rasa
rasa train
rasa shell
``` 
This will enable us to chat with the bot. 
You can also use the interactive mode to test the bot while improving it at the same time by runnning this instead
```
cd rasa
rasa train
rasa interactive
```
### Using webchat template
Let's try using the template we have integrated using Flask. We are going to create two different models for both Rasa NLU and Rasa Core.

To train Rasa NLU:
```
cd rasa
python -m rasa_nlu.train -d data/nlu.md -c config.yml --project profiler --fixed_model_name nlu
```
To train Rasa Core:
```
cd rasa
python -m rasa_core.train -s data/stories.md -d domain.yml -o models/dialogue
```
These commands will generate two folders under models. You will have to train everytime you made changes to like the domain, nlu or stories files.

After training, run the rasa server:
```
python -m rasa.server
```
After that, you can either open up the `index.html` file manually or run flask by doing:
```
python app.py
```

## Using Custom Actions
Let's define a custom action. We will have a custom action that saves name to a slot which is bascially a storage for that last during that particular session. Of course name can be saved straight away using slots but sometimes it is not as accurate. So we are going to come up with a custom action to be called after answering name. 

1. To use the custom action server, we will need to configure it! Uncomment these two lines in `endpoints.yml` file:
    ```
    action_endpoint:
    url: "http://localhost:5055/webhook"
    ```
2. Insert `action_endpoint=EndpointConfig('http://localhost:5055/webhook')` in `server.py` file so that server knows where the enpoint is for the action server. (This should already be there from earlier on)

3. We will need the bot to ask the name. So let's change the `utter_intro` to this:
    ```
    templates:
      utter_intro:
      - text: "Hi! My name is Rsykoss :) Nice to meet you! What is your name?"
    ```
4. After the question is asked, the person replying will most likely say his/her name. So we will need an intent called `say_name`. Include this in the `domain.yml` file.
    ```
    intents:
      - say_name
    ```
    Then, we will still need to give examples of how the person can say the intent. We need to list this in the `nlu.md` file. An example of how the nlu.md file will look for `say_name` intent is this:
    ```
    ## intent:say_name
    - I'm [John](name)
    - [aladdin](name)
    - my name is [jack](name)
    - hi i am [xiao ming](name)
    - call me [alice](name)
    - i'm [paul](name)
    - hi i'm [Nick](name)
    - hello call me [Jia Ling](name)
    - i am [mary ong](name)
    - hey call me [Zara](name)
    - yo i am [renee](name)
    - i am [wei ling](name)
    - you can address me as [mr tan](name)
    - i am [aaron](name)
    - [carl](name)
    - i am [paula](name)
    - [shyann](name)
    - im [mike](name)
    - you can call me [rsykoss](name)
    - im [jie ying](name)
    - call me [michelle obama](name)
    - im [elizabeth](name)
    - I'm [keith](name)
    - [jay](name)
    - [chris](name)
    - [sy](name)
    - [yy](name)
    - [jj](name)
    - [Michael](name)
    - [liza peet](name)
    - [dunkel hills](name)
    - [Clarisa Attwood](name)
    - [Melanie Bonavita](name)
    - [Jia Hui](name)
    - [Zhi Hao](name)
    - [Yi Ling](name)
    - [Grazyna Schiel](name)
    - [Hyacinth](name)
    - [Alexis](name)
    - [Gisela](name)
    ```
    The more relevant example you provide, the more accurate the intent detection will be. 

5. After the question is asked, we will bring in the action here so that whatever you have typed can be saved. So let's create an action called `action_save_name`. Define this in `domain.yml` file first.
    ```
    actions:
    - action_save_name
    .....
    ```
6. When you are saving the name, you will need to store it in something called `slots`! This is also defined in the `domain.yml` file like this:
    ```
    slots:
      name:
        initial_value: Human
        type: text
    ```
    In this case, we are saving it as a variable called "name" and the initial value will be Human and the type will be text. 

7. After the bot saves the name, it should say something like "Hi! Nice to meet you {name}!". So let's create an utterance called `utter_name`. In this case, {name} will be automatically be replaced with the slot "name".
    ```
    actions:
    - utter_name
    ..............

    templates:
      utter_name:
      - text: "Hi! Nice to meet you {name}!"
      .......... 
    ```
8. Now that we have defined all the required things except the action itself, we will also have to modify the stories so that the path will go through the action. So the flow is: when intent is start --> bot will introduce himself and ask for name --> then you reply with intent of saying name --> bot saves this name --> then the bot uses the saved name to reply back --> then the story continues. Insert the following to the start of all the paths like this.

    ```
    ## happy path
    * get_started
      - utter_intro
    * say_name
      - action_save_name
      - utter_name
    * greet
      - utter_greet
    * mood_great
      - utter_happy

    ```
9. Now, let's work on custom actions in `actions.py` file in rasa.


## Deploy using ngrok


