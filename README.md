YOLOdice API Client for Ruby
============================

[YOLOdice](https://yolodice.com) is a simple online Bitcoin game you can play against the house. The game relies on a pseudorandom number generator that returns bet results used to determine if bets placed by players win or lose.

This Ruby library contains a simple client that connects to the YOLOdice API endpoint:

* client connects over secure SSL over TCP transport layer,
* supports authentication,
* handles any server-side methods via `method_missing` mechanism.

Additional resources:

* [dev.yolodice.com](https://dev.yolodice.com) Official YOLOdice API documentation,
* [yolodice.com](https://yolodice.com) The YOLOdice site (take a look at [FAQ](https://yolodice.com/#faq)).

## Installation

    gem install yolodice-client


## Usage

Require the gem in your scripts:

    require 'yolodice_client'


## Generating API keys

YOLOdice API requires authentication for most of it's methods. A Bitcoin key/address will be required to setup the API key and authenticate. Here is one way to do this:

1. Generate an Bitcoin public and private key:

    require 'bitcoin'

    btc_key = Bitcoin::Key.generate
    auth_key = btc_key.to_base58  # this is your secret code, store it in a secure place
    auth_addr = btc_key.addr      # paste this in your YD settings as a new key

2. Go to [YOLOdice account Settings](https://yolodice.com/#settings), create a new key and paste the `auth_addr` generated above. Set permissions as you wish.
3. Use `auth_key` in your code to authenticate.

Just a quick note &mdash; this address is used ONLY to authenticate. No coins will be ever sent to it.


## Connecting

    yd = YolodiceClient.new
    yd.connect
    yd.authenticate auth_key

It's important to authenticate immediately after connecting. Otherwise the connection will be closed by the server.

The client automatically sends a `ping` requests to the server every 30 seconds to prevent the connection from timing out.


## Logging and debugging

To preview the actuall messages sent back and forth you could provide your own logger object and set log level to `DEBUG` like this:

    yd = YolodiceClient.new
    logger = Logger.new STDERR
    logger.level = Logger::DEBUG
    yd.logger = logger

    yd.connect
    yd.authenticate auth_key

This would result in the output similar to this:

    DEBUG -- : Connecting to api.yolodice.dev:4444 over SSL
    INFO  -- : Connected to api.yolodice.dev:4444
    DEBUG -- : Listening thread started
    DEBUG -- : Pinging thread started
    DEBUG -- : Calling remote method generate_auth_challenge()
    DEBUG -- : >>> {"id":1,"method":"generate_auth_challenge"}
    DEBUG -- : <<< {"id":1,"result":"yd_login_26vEyvUUdgUy"}

    DEBUG -- : Calling remote method auth_by_address({:address=>"n3kmufwdR3Zzgk3k6NYeeLBxB9SpHKe5Tc", :signature=>"IB5ITZHQZoApdXhUMGFFZ9AG4OtTw85jdaPMSVYNpOayEAG5LK9bsPhtCjwPEjDy/YDHqKk6gf1+aLzg0B63Qfk="})
    DEBUG -- : >>> {"id":2,"method":"auth_by_address","params":{"address":"n3kmufwdR3Zzgk3k6NYeeLBxB9SpHKe5Tc","signature":"IB5ITZHQZoApdXhUMGFFZ9AG4OtTw85jdaPMSVYNpOayEAG5LK9bsPhtCjwPEjDy/YDHqKk6gf1+aLzg0B63Qfk="}}
    DEBUG -- : <<< {"id":2,"result":{"id":1,"name":"sdafasfuiafu","created_at":1470085899.0356,"roles":["admin"]}}

## Conventions, return values, errors

By default whenever any Bitcoin amount is sent or received from the server, it is passed as an Integer with the amount of satoshis. Using this convention 1 BTC would be represented as an integer value or `100_000_000`.

Whenever server responds with an Object, this client returns a Hash with keys being Strings.

There are two error classes:

* `YolodiceClient::Error` that inherits from `StandardError` and is used for errors thrown by the client itself,
* `YolodiceClient::RemoteError` that inherits from `StandardError` that is used to pass errors from the remote server,
* any errors from underlaying `TCPSocket` or `SSLSocket` are passed through.

`RemoteError` has two extra attributes: `code` and `data` that are mapped to values in the error object returned from the server.

## Another example

Here is a script that connects to the server, authenticates, fetches user data, rolls a few 50% chance bets and reads user data again (make sure to use your own credential):

    require 'yolodice_client'
    require 'pp'

    auth_key = 'cPFVHENWNjs5UKNXXynDSWiRkBEph8hcrjHKkXK5SW9QHxx7i4jC'
    yd = YolodiceClient.new
    yd.connect
    user = yd.authenticate auth_key
    user_data = yd.read_user_data selector: {id: user['id']}
    puts "Your account balance is: #{user_data['balance']} satoshis."
    10.times do
      b = yd.create_bet attrs: {amount: 100, range: 'lo', target: 500000}
      puts "Bet profit: #{b['profit']}"
    end
    user_data = yd.read_user_data selector: {id: user['id']}
    puts "Your account balance is: #{user_data['balance']} satoshis."