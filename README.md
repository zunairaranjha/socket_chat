<<<<<<< HEAD
# socket_chat
no
=======
# Basic Redis chat application demo in Ruby

A basic chat application built with Ruby on Rails with ActionCable and Redis.

<img src="https://github.com/redis-developer/basic-redis-chat-demo-ruby/raw/master/app/assets/images/main_screenshot1.png" width="49%"></a>
<img src="https://github.com/redis-developer/basic-redis-chat-demo-ruby/raw/master/app/assets/images/main_screenshot2.png" width="49%"></a>

# Overview video

Here's a short video that explains the project and how it uses Redis:

[![Watch the video on YouTube](https://github.com/redis-developer/basic-redis-chat-demo-ruby/raw/master/public/YTThumbnail.png)](https://www.youtube.com/watch?v=miK7xDkDXF0)

## Tech Stack

_Ruby on Rails_, _Socket_ (_ActionCable_), _Redis_

## How it works?

### Database Schema

#### User

```Ruby
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  validates :username, uniqueness: true, presence: true

  has_many :room_messages,
           dependent: :destroy

  def online?
    Redis.new.get("user_#{self.id}_online").present?
  end

  def need_logout?
    Redis.new.get("logout_#{self.id}").present?
  end

  def self.online_users
    Redis.new.hgetall('online_users')
  end
end
```

#### ChatRoom

```Ruby
class Room < ApplicationRecord
  has_many :room_messages, dependent: :destroy,
                           inverse_of: :room
  has_many :users, through: :room_messages
end
```

#### ChatRoomMessage

```Ruby
class RoomMessage < ApplicationRecord
  validates :message, presence: true

  belongs_to :room, inverse_of: :room_messages
  belongs_to :user

  before_save :save_username

  def self.all_messages
    Redis.new.hgetall('room_messages')
  end

  private

  def save_username
    self.username = user.username
  end
end
```

### Initialization

For simplicity, a key with **total_users** value is checked: if it does not exist, we fill the Redis database with initial data.
`EXISTS total_users` (checks if the key exists)

The demo data initialization is handled in multiple steps:

**Creating of demo users:**
We create a new user id: `INCR total_users`. Then we set a user ID lookup key by user name: **_e.g._** `SET username:nick user:1`. And finally, the rest of the data is written to the hash set: **_e.g._** `HSET user:1 username "nick" password "bcrypt_hashed_password"`.

Additionally, each user is added to the default "General" room. For handling rooms for each user, we have a set that holds the room ids. Here's an example command of how to add the room: **_e.g._** `SADD user:1:rooms "0"`.

**Populate private messages between users.**
At first, private rooms are created: if a private room needs to be established, for each user a room id: `room:1:2` is generated, where numbers correspond to the user ids in ascending order.

**_E.g._** Create a private room between 2 users: `SADD user:1:rooms 1:2` and `SADD user:2:rooms 1:2`.

Then we add messages to this room by writing to a sorted set:

**_E.g._** `ZADD room:1:2 1615480369 "{'from': 1, 'date': 1615480369, 'message': 'Hello', 'roomId': '1:2'}"`.

We use a stringified _JSON_ for keeping the message structure and simplify the implementation details for this demo-app.

**Populate the "General" room with messages.** Messages are added to the sorted set with id of the "General" room: `room:0`

### Registration

![How it works](app/assets/images/main_screenshot1.png)

Redis is used mainly as a database to keep the user/messages data and for sending messages between connected servers.

#### How the data is stored:

- The chat data is stored in various keys and various data types.
  - User data is stored in a hash set where each user entry contains the next values:
    - `username`: unique user name;
    - `password`: hashed password

* User hash set is accessed by key `user:{userId}`. The data for it stored with `HSET key field data`. User id is calculated by incrementing the `total_users`.

  - E.g `INCR total_users`

* Username is stored as a separate key (`username:{username}`) which returns the userId for quicker access.
  - E.g `SET username:Alex 4`

#### How the data is accessed:

- **Get User** `HGETALL user:{id}`

  - E.g `HGETALL user:2`, where we get data for the user with id: 2.

- **Online users:** will return ids of users which are online
  - E.g `SMEMBERS online_users`

#### Code Example: Create User Session

```Ruby
  def create
    user = User.find(params[:id])
    if user.present? && user.valid_password?(params[:password])
      sign_in(user)
      Redis.new.del("logout_#{current_user.id}")
      broadcast(true)
      send_connect_message("#{current_user.username} connected") unless current_user.online?
      redirect_to root_path
    else
      redirect_to new_user_session_path
    end
  end
```

### Rooms

![How it works](app/assets/images/main_screenshot2.png)

#### How the data is stored:

Each user has a set of rooms associated with them.

**Rooms** are sorted sets which contains messages where score is the timestamp for each message. Each room has a name associated with it.

- Rooms which user belongs too are stored at `user:{userId}:rooms` as a set of room ids.

  - E.g `SADD user:Alex:rooms 1`

- Set room name: `SET room:{roomId}:name {name}`
  - E.g `SET room:1:name General`

#### How the data is accessed:

- **Get room name** `GET room:{roomId}:name`.

  - E. g `GET room:0:name`. This should return "General"

- **Get room ids of a user:** `SMEMBERS user:{id}:rooms`.
  - E. g `SMEMBERS user:2:rooms`. This will return IDs of rooms for user with ID: 2

#### Code Example: Get all My Rooms

```Ruby
  def load_entities
    @rooms = Room.joins(:users).where(users: { id: current_user.id }).uniq
    @room = params[:id].present? ? Room.find(params[:id]) : Room.first
  end
```

### Messages

#### Pub/sub

After initialization, a pub/sub subscription is created: `SUBSCRIBE MESSAGES`. At the same time, each server instance will run a listener on a message on this channel to receive real-time updates.

Again, for simplicity, each message is serialized to **_JSON_**, which we parse and then handle in the same manner, as WebSocket messages.

Pub/sub allows connecting multiple servers written in different platforms without taking into consideration the implementation detail of each server.

#### How the data is stored:

- Messages are stored at `room:{roomId}` key in a sorted set (as mentioned above). They are added with `ZADD room:{roomId} {timestamp} {message}` command. Message is serialized to an app-specific JSON string.
  - E.g `ZADD room:0 1617197047 { "From": "2", "Date": 1617197047, "Message": "Hello", "RoomId": "1:2" }`

#### How the data is accessed:

- **Get list of messages** `ZREVRANGE room:{roomId} {offset_start} {offset_end}`.
  - E.g `ZREVRANGE room:1:2 0 50` will return 50 messages with 0 offsets for the private room between users with IDs 1 and 2.

#### Code Example: Create Message

```Ruby
  def create
    @room_message = RoomMessage.create user: current_user,
                                       room: @room,
                                       message: params.dig(:room_message, :message)
    Redis.new.hset('room_messages', @room_message.id, @room_message.message)

    ActionCable.server.broadcast 'room', @room_message
  end
```

### Session handling

The chat server works as a basic _REST_ API which involves keeping the session and handling the user state in the chat rooms (besides the WebSocket/real-time part).

When a WebSocket/real-time server is instantiated, which listens for the next events:

**Connection**. A new user is connected. At this point, a user ID is captured and saved to the session (which is cached in Redis). Note, that session caching is language/library-specific and it's used here purely for persistence and maintaining the state between server reloads.

A global set with `online_users` key is used for keeping the online state for each user. So on a new connection, a user ID is written to that set:

**E.g.** `SADD online_users 1` (We add user with id 1 to the set **online_users**).

After that, a message is broadcasted to the clients to notify them that a new user is joined the chat.

**Disconnect**. It works similarly to the connection event, except we need to remove the user for **online_users** set and notify the clients: `SREM online_users 1` (makes user with id 1 offline).

**Message**. A user sends a message, and it needs to be broadcasted to the other clients. The pub/sub allows us also to broadcast this message to all server instances which are connected to this Redis:

`PUBLISH message "{'serverId': 4132, 'type':'message', 'data': {'from': 1, 'date': 1615480369, 'message': 'Hello', 'roomId': '1:2'}}"`

Note we send additional data related to the type of the message and the server id. Server id is used to discard the messages by the server instance which sends them since it is connected to the same `MESSAGES` channel.

`type` field of the serialized JSON corresponds to the real-time method we use for real-time communication (connect/disconnect/message).

`data` is method-specific information. In the example above it's related to the new message.

## How to run it locally?

### Prerequisites

- Ruby - v2.6.2
- Rails - v5.2.4.5
- NPM - v7.6.0

#### Run

```sh
# Set proper data inside
  Open file config/application.yml and set value to keys:
- REDIS_URL: Redis server URI
- USER: Redis username
- PASSWORD: Redis password

# Run commands in your terminal
rails db:create
rails db:migrate
rails db:seed
```

#### Run the app

```sh
rails s
```

#### Put this link in your browser(localhost example)

```sh
http://localhost:3000
```

## Try it out

#### Deploy to Heroku

<p>
  <a href="https://heroku.com/deploy" target="_blank">
      <img src="https://www.herokucdn.com/deploy/button.svg" alt="Deploy to Heorku" />
  </a>
</p>
<p>
  <a href="https://deploy.cloud.run/?git_repo=https://github.com/redis-developer/basic-redis-chat-demo-ruby" target="_blank">
     <img src="https://deploy.cloud.run/button.svg" alt="Run on Google Cloud" width="150px"/>
 </a>
</p>
# App
# App
# App
# App
>>>>>>> 7e83e56 (socket)
