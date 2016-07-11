# README

This is a transcription of a demo that DDH did of ActionCable.

https://medium.com/@dhh/rails-5-action-cable-demo-8bba4ccfc55e#.q92qfr3c2

This is part of a explaination of how to deploy ActionCable applications to heroku.

## Creating the app walkthrough

Create app, basic controller and model:

```
$ rails new chatapp
$ cd chatapp
$ rails g controller rooms show
$ rails g model message content:text
```

app/views/controllers/rooms_controller.rb:

```
class RoomsController < ApplicationController
  def show
    @messages = Message.all
  end
end
```

app/views/messages/_message.html.erb

```
<div class="message">
  <p><%= message.content %></p>
</div>
```

app/views/room/show.html.erb

```
<h1>Chat room</h1>

<div id="messages">
  <%= render @messages %>
</div>
```

config/routes.rb

```
  root 'rooms#show'

  mount ActionCable.server => '/cable'
```

### Create ActionCable channel

```
$ rails g channel room speak
```


app/assets/javascripts/channels/room.coffee

```
  received: (data) ->
    $('#messages').append data['message']

  speak: (message) ->
    @perform 'speak', message: message
```


room_channel.rb

```
  def subscribed
    stream_from 'room_channel'
  end

  def speak(data)
    ActionCable.server.broadcast 'room_channel', message: data['message']
  end
```


app/views/rooms/show.html.erb

```
<form>
  <label>Say Something</label><br/>
  <input type="text" data-behavior="room_speaker"/>
</form>
```


app/assets/javascripts/channels/room.coffee

```
$(document).on 'keypress', '[data-behavior~=room_speaker]', (event) ->
  if event.keyCode == 13
    App.room.speak event.target.value
    event.target.value = ''
    event.preventDefault()
```


app/channels/room_channel.rb

```
  def speak(data)
    Message.create! content: data['message']
  end
```

### Create the ActionJob to do the background sending

```
$ rails g job message_broadcast
```

Trigger the job in the after create commit hook.

app/models/message.rb
```
  after_create_commit do
    MessageBroadcastJob.perform_later self
  end
```

The job itself

```
class MessageBroadcastJob < ApplicationJob
  queue_as :default

  def perform(message)
    ActionCable.server.broadcast 'room_channel', message: render_message( message )
  end

  private

  def render_message( message )
    ApplicationController.renderer.render partial: 'messages/message', locals: { message: message }
  end
end
```


