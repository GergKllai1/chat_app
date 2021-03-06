# Basic chat app using Rails ActionCable functionality

##### Written by [Adi](https://github.com/kianaditya) and [Greg](https://github.com/GergKllai1)

We will be building a basic chat app to learn about the [ActionCable functionality](https://guides.rubyonrails.org/action_cable_overview.html) which uses [websockets](https://en.wikipedia.org/wiki/WebSocket) protocol.

Our App will have following features:

- User Authentication
- Push notifications in chat windows
- List of chats and list of available users

The final app will look something like this:


![](gif.gif)

`Disclaimer: This guide assumes a basic familiarity with rails development, user authentication, and we expect you to follow the TDD way of iterative development.`

## Basic setup

### Scaffold a rails application

First scaffold basic rails app

`rails new chat_app --database=postgresql --skip-test --skip-bundle`

Clean up the usual files.

### Set up testing

Set up [Cucumber](https://github.com/cucumber/cucumber-rails) for testing.
We set up a feature file to make sure two users can chat with each other, and others cannot access the chat.

```gherkin
# features/live_chat_functionality.feature

@javascript
Feature: LiveChat allows users to exchange messages
    As a user
    In order to chat with other users
    I want a live chat feature

    Background:
        Given the following users exist
            | email             |
            | user-1@random.com |
            | user-2@random.com |
            | user-3@random.com |

        And I am logged in as "user-1@random.com"
        And I visit the site
        And I click on "user-2@random.com"

    Scenario: Users can exchange messages
        Given I fill in "Hello!" in "message_text"
        And I click on 'Send'
        And I open a new window
        And I log in as "user-2@random.com"
        And I visit the site
        And I click on "Chat with user-1@random.com"
        Then I should see "Chat with: user-1@random.com"
        Then I should see "Hello!"
        And I fill in "Hello there!" in "message_text"
        And I click on 'Send'
        And I switch to window 1
        And I should see "Hello there!"

    Scenario: Other users cannot see the chat

        Given I open a new window
        And I log in as "user-3@random.com"
        And I visit the site
        Then I should not see "join"
```

This one is tricky to test, because we have to manage multiple users logged into multiple windows. Makes it even more fun when we will eventually get it to work!

On the top you can see `@javascript` which means that we will use javascript during our testing. To enable that we need to add chromedriver and configure it.

Add the following gems and bundle:

```ruby
  gem 'chromedriver-helper'
  gem 'selenium-webdriver'
```

And add following chrome options to cucumber `env.rb`

```ruby
#features/support/env.rb
Cucumber::Rails::Database.javascript_strategy = :truncation

Chromedriver.set_version '2.42'

chrome_options = %w[no-sandbox disable-popup-blocking disable-infobars]

chrome_options << 'headless'

Capybara.register_driver :selenium do |app|
    options = Selenium::WebDriver::Chrome::Options.new(
        args: chrome_options
)
Capybara::Selenium::Driver.new(
        app,
        browser: :chrome,
        options: options
)
end
```

Create `basic_steps.rb` and `assertion_steps.rb` step definition files and store the step definitions appropriately.

Let's start by running cucumber. Fill in the step definition as:

```ruby
#features/step_definitions/basic_steps.rb
Given("the following users exist") do |table|
    table.hashes.each do |user|
        create(:user, user)
    end
end
  
Given("I( am) logged/log in as {string}") do |email|
    user = User.find_by(email: email)
    login_as(user, scope: :user)
end
  
Given("I visit the site") do
    visit root_path
end
  
Given("I open a new window") do
    window = open_new_window
    switch_to_window(window)
end

Given("I click on {string}") do |element|
    click_on element
end
  
Given("I fill in {string} in {string}") do |value, element|
    fill_in element, with: value
end
  
Given("I switch to window {int}") do |index|
    switch_to_window(windows[index - 1])
end
```

```ruby
#features/step_definitions/assertion_steps.rb
Then("I should see {string}") do |content|
    expect(page).to have_content content
end

Then("I should not see {string}") do |content|
    expect(page).not_to have_content content
end
```

To get it to work, we need to add the line `World(FactoryBot::Syntax::Methods)` to our `features/support/env.rb` file.

It will now complain about lack of users in our code.

### Set up Devise

Set up [devise](https://github.com/plataformatec/devise) which will be used for user creation/authentication.

After the set up, make sure you have a **User** model and a working factory for user.

Now that we have users in the system, we need to add chat and messaging functionality.

### Generate models

Now comes the fun part!
We will need two models other then the **User** model generated by devise: **Chat** and **Message** .

**Chat** model will be empty and **Message** model will have a 'text' column with string datatype.

Use the following commands:

`rails g model Chat` and `rails g model Message text:string`

to generate each.

### Set up associations

Now is the time to add associations!

1. **Chats** and **Messages** have `has_many` & `belongs_to` associations.
2. **Users** and **Messages** have `has_many` & `belongs-to` associations.
3. **Chats** and **Users** have `has_many_and_belongs_to` association(!).

To do this run the following commands:

```rails
rails g migration CreateJoinTableUserChat user chat
rails g migration AddUserRefToMessages user:references
rails g migration AddChatRefToMessages chat:references
rails db:migrate
```

And add the associations:

```ruby
#app/models/chat.rb
class Chat < ApplicationRecord
  has_and_belongs_to_many :users
  has_many :messages
end
```

```ruby
#app/models/message.rb
class Message < ApplicationRecord
  belongs_to :chat
  belongs_to :user
end
```

```ruby
#app/models/user.rb
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable, :trackable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable
  has_and_belongs_to_many :chats
  has_many :messages
end
```

### Set up Chat controller and starter views

Set up chat and messages controllers with basic index and show action. Create `chat_controller.rb` with the following code:

```ruby
#app/controllers/chat_controller.rb
class ChatController < ApplicationController
  def index
    @chats = current_user.chats
  end

  def show
      @messages = Message.where(chat_id: params[:id])
    end
end
```

Set up routes in `routes.rb`. We will use `chat#index` as root path.

```ruby
#config/routes.rb
Rails.application.routes.draw do
  root controller: :chat, action: :index
  devise_for :users
  resources :chat, only: [:index, :show]
end
```

And to finish off add 3 views to start with. First we add a navbar partial so that we are able to log in and out:

```ruby
#app/views/partials/_navbar.html.haml
= link_to 'Home', root_path
-unless user_signed_in?
  = link_to 'Register', new_user_registration_path
  = link_to 'Login', new_user_session_path
- else
  = link_to 'Logout', destroy_user_session_path, { method: :delete }
  %span= current_user.email
```

Render this partial in the `app/layouts/application.html.haml` After that we add the index view for chat:

```ruby
#app/view/chat/index.html.haml
-@chats.each do |chat|
  = link_to "Chat between #{chat.users.first.email} and #{chat.users.second.email}", chat_path(chat.id)
```

To finish up this we need to add the chat show page as well:

```ruby
#app/view/chat/show.html.haml
- @messages.each do |message|
  %p= "#{message.user.email} says: #{message.text}"
```

To see this in action we need to add some data, the easiest way is through the seed file. Let's add the following code to it:

```Ruby
#db/seed.rb
Message.destroy_all
Chat.destroy_all
User.destroy_all

user1 = User.create(email: 'user1@mail.com', password:'password')
user2 = User.create(email: 'user2@mail.com', password:'password')
user3 = User.create(email: 'user3@mail.com', password:'password')

chat_1 = Chat.create()
chat_1.users << user1
chat_1.users << user2

chat_2 = Chat.create()
chat_2.users << user1
chat_2.users << user3
```

Now if you run `rails db:seed` and `rails s` you should be able to login and see 2 chat links for user1.

### Create Message controller

Now we should continue by adding a `message_controller` with only create action. For this implementation we will use `chat#show` to render the message form.

Add the following file:

```ruby
#app/controllers/message_controller
class MessageController < ApplicationController
  def create
    message = current_user.messages.new(message_params)
    chat = Chat.find_by_id(message_params[:chat_id])
    if message.save
      redirect_to chat_path(chat)
    end
  end

  private

  def message_params
    params.require(:message).permit(:text, :chat_id)
  end
end
```

The `message = current_user.messages.new(message_params)` uses the `has_many` association to create a message connected to the current user already, which makes for a drier code.

Add the path to the `routes.rb`:

```ruby
#config/routes.rb
Rails.application.routes.draw do
  root controller: :chat, action: :index
  devise_for :users
  resources :chat, only: [:index, :show]
  resources :message, only: [:create]
end
```

And finally add the form to the `chat#show` view:

```Haml
-@messages.each do |message|
  %p= "#{message.user.email} says: #{message.text}"

#message_window
= form_with scope: :message, url: message_index_path, id: :chat_form do | form |
  %p
    = form.label 'Send Message'
    %br/
    = form.text_field :text
    = form.hidden_field :chat_id, value: @chat.id,id: :chat_id
  %p
    = form.submit "Send"
```

`#message_window` will be the container where will display the messages we broadcast.

We need to pass down the `chat_id` using a hidden field so the message can be connected to the chat.

Now we have a chat with 2 users, and a form where we can create messages. We render those messages in the `chat#show`, and when you type you see them appearing. This is because rails redirects us to that page and fetches the information again. That is not the behaviour we are looking for, we need to show them appear dynamically. This is where ActionCable comes into play.

## Set up ActionCable

### Redis setup

Redis is a data store that supports the PubSub messaging pattern and one that the ActionCable implementation makes use of.

Install Redis using brew

`$ brew install redis`

Start the Redis server as a service

`$ brew services start redis`

You should also include the redis gem in your application by adding:

`gem 'redis'`

Follow up by running:

`bundle`

After that you have to configure redis. Add the following code to `config/cable.yml`

````ruby
redis: &redis
  adapter: redis
  url: redis://localhost:6379/1
production: *redis
development: *redis
test: *redis```

````

### Add ActionCalbe Channels

Add ActionCable engine to your `routes.rb`

```ruby
Rails.application.routes.draw do
  mount ActionCable.server => '/cable'
end
```

Use a generator to scaffold a channel

`rails g channel chat`

This will generate couple of files for us that we need to configure. Let's follow the flow. First we add the the `ActionCable.server.broadcast` method to the `message_controller.rb`. Change the file so that it looks like this:

```ruby
#app/controllers/chat_controller.rb
class MessageController < ApplicationController
  before_action :authenticate_user!
  def create
    message = current_user.messages.new(message_params)
    chat = Chat.find_by_id(message_params[:chat_id])
    if message.save
      ActionCable.server.broadcast("chat_channel_#{chat.id}", message: message.text, from: current_user.email)
      head :ok
    end
  end

  private

  def message_params
    params.require(:message).permit(:text, :chat_id)
  end
end
```

We are using `chat.id` to specify the chat channel we want to broadcast to, use `message.text` as the message body, and `current_user.email` to specify who is the sender.

We are creating unique chat channel for every pair of users, and we need a method to subscribe to specific channel based on the logged in user and the partner selected.

Time to update javascript file that was generated for us:

```javascript
//app/assets/javascripts/channels/chat.js

document.addEventListener('turbolinks:load', () =>{
    let chatForm = document.getElementById('chat_form');
    if (chatForm) {
        const chatId = document.getElementById('chat_id').value
        const currentUserEmail = document.getElementById('current_user_email').value
        App.notifications = App.cable.subscriptions.create({
            channel: "ChatChannel", chat_id: chatId
          }, {
                container() {
                    const container = document.getElementById('message_window');
                    return container;
                },
                connected() {
                    console.log(`Connected to message:chat_${chatId}`);
                },
                disconnected() {
                    console.log('Disconneced');
                },
                received(data) {
                    let node = document.createElement('p');
                    node.className = data.from === currentUserEmail ? 'send-message' : 'receive-message'
                    node.innerText = `${data.message}`;
                    this.container().appendChild(node);
                },
            }
          );  
    };
});
```

This needs some explanation!

At the core of the file is the `App.cable.subscriptions.create` command, which creates notifications and updates the page through `received(data)` function by adding an HTML element to the page everytime a new message is received.

This is controlled by identifying `chatId` which we source from our chat form.

Now, in order to get that `chatId` we have to make sure the `chatForm` exists on the view page, so we make sure page is loaded and form exists by wrapping the whole thing in event listeners.

The last piece of the puzzle is to update the `ChatChannel` settings in ruby like so:

```ruby
#app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  def subscribed
    stream_from channel_identifier
  end
  
  def unsubscribed
    # Any cleanup needed when channel is unsubscribed
  end

  private

  def channel_identifier
    identifier = params[:chat_id]
    "chat_channel_#{identifier}"
  end
end
```

This will kick in once we ran the `App.cable.subscriptions.create` command and will stream from the channel we constructed with `channel_identifier`.

## Connecting the dots

Now we have a basic chat application but we don't have a way for users to create chats with users they are not chatting with. We used the seed file so far.

To do that we need to add a `create` action for `Chat`. It would be also great, to render these elements conditionally, so that we can only create chats with users that we are not in chat with.

The following code will be a first iteration it will be up to you to refactor it. Modify the `chat_controller.rb` and the chat views accordingly:

```ruby
#app/controllers/chat_controller.rb
class ChatController < ApplicationController
  def index
    if user_signed_in?
      @chats = current_user.chats
      @users = get_users(get_existing_chat_users)
    end
  end

  def show
    @chat = Chat.find(params[:id])
    @messages = Message.where(chat_id: params[:id])
    @chat_partner = @chat.users.select { |user| user != current_user }.first
  end

  def create
    user = User.find_by_id(params[:user])
    chat = Chat.create()
    chat.users << [current_user, user]
    redirect_to chat_path(chat)
  end

  private

  def get_existing_chat_users
    @chats.map{ |chat| chat.users.select{ |user| user }}.flatten
  end

  def get_users(existing_chat_users)
    User.all.select{ |user| !existing_chat_users.include?(user) }
  end
  
end
```

```Haml
#app/views/chat/index.html.haml
-if user_signed_in?
  .main
    .chats
      %h4 Active chats:
      - @chats.each do |chat|
        - chat.users.each do |user|
          - unless user === current_user
            %p= link_to "Chat with #{user.email}", chat_path(chat.id), { class: 'links join'}
    .chats
      %h4 Start a new chat with:
      - @users.each do |user|
        - unless user == current_user
          = link_to "#{user.email}", chat_index_path(user: user), { class: 'links join' , method: :post }
          %br/
```

`app/views/chat/show.html.haml`

```Haml
.main
  .chats
    ="Chat with: #{@chat_partner.email}"
    - @messages.each do |message|
      - if message.user.email == current_user.email
        %p.send-message= message.text
      - else
        %p.receive-message = message.text
    #message_window.message-container
    = form_with scope: :message, url: message_index_path, id: :chat_form do | form |
      = form.hidden_field :chat_id, value: @chat.id, id: :chat_id
      = form.hidden_field :current_user_email, value: current_user.email, id: :current_user_email

      %p
        = form.label 'Send Message'
        %br/
        = form.text_field :text
      %p
        = form.submit "Send"
```

With this we arrived to the end of our planned functionlity. By running cucumber we should see all of our tests passing.

### Conditional Styling

As you might have noticed we added some classes to the layout which we can use for styling purposes. The most important ones were `send-message` and `recieve-message`  in `app/assets/javascripts/channels/chat.js` and in show `app/views/chat/show.html.haml`. We used some conditionals in ruby and javascript to differenciate between the send and recieved messages. Below you can find our simple styling using only css, you can add of course yours:

```css
/* app/assets/stylesheets/application.css */

body {
  font-size: 120%;
  background-color: #F2F6F7;
  margin: 0;
  letter-spacing: 1px;
}
.main {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.chats {
  background: #008AAD;
  color: white;
  border: 1px solid black;
  width: 60%;
  text-align: center;
  margin: 10px;
  padding: 12px;
  border-radius: 10px;
  display: flex;
  flex-direction: column;
}

.navbar {
  background: #008AAD;
  color: white;
  height: 50px;
}

.links {
  text-decoration: none;
  color: white;
  margin: 15px;
}

.send-message{
  background-color:darkgreen;
  width: 30%;
  border-radius: 20px;
  margin: 20px;
  padding: 10px;
  align-self: flex-end;
}

.receive-message{
  background-color: rgb(79, 79, 79);
  width: 30%;
  border-radius: 20px;
  margin: 20px;
  padding: 10px;
  align-self: flex-start;
}

.message-container{
  display: flex;
  flex-direction: column;
}

.join {
  border:2px black solid;
  background: dimgray;
  padding: 10px;
  border-radius: 10px;
}

.join:hover {
  background: darkgreen;
}
```

## Conclusion

At this point we should have a simple functioning chat application, with instantaneous feedback, tested with cucumber. We left a lot of room for improvements, for instance to add another channel for chat_windows so users get instant feedback when someone starts a chat with them.

We hope that you have learned something during this guide, and enjoyed it as much as we enjoyed writing it :).
