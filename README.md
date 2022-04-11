# NAME

Telegram::JsonAPI - Telegram TDLib's JSON API

# SYNOPSIS

    use Telegram::JsonAPI qw(:all);

    td_start_log();

    my $res = td_execute('{"@type": "setLogVerbosityLevel", "new_verbosity_level": 1, "@extra": 1.01234}');
    # got $res = '{"@type":"ok","@extra":1.01234}';

    td_poll_log(sub {
      my($verbosity, $msg) = @_;
      print "got log ($verbosity) $msg\n";
    });
    # this will clear all the buffered log once.

    my $client_id = td_create_client_id();
    td_send($client_id, '{"@type": "getAuthorizationState", "@extra": 1.01234}');
    # start loggin progress

    while(1) {
      my $msg = decode_json(td_receive(.1));
      # Wait for a JSON message for at most 0.1 second.
      # There will be a `@client_id` field, which contains $client_id.
      # If you have more than one client id simutaneously, you can distinguish them by this field.
      given( $msg->{'@type'} ) {
        when('updateAuthorizationState') {
          ...
        }
        ...
      }
    }

    td_stop_log();

# DESCRIPTION

This module integrate [Telegram](https://telegram.org/)'s TDLib [JSON API](https://core.telegram.org/tdlib/docs/td__json__client_8h.html).
which is used to implement Telegram client app. The difference between an app and a bot is that an app will act as an normal user.
And you need to authenticate it with a phone number.

## EXPORT

None by default.

With tag `:all`, there are

- $client\_id = td\_create\_client\_id()

    Returns an opaque identifier of a new TDLib instance. The TDLib instance will not send updates until the first request is sent to it.

- td\_send($client\_id, $json\_request)

    Sends request to the TDLib client.

- $json\_message = td\_receive($timeout)

    Receives incoming updates and request responses.

- $json\_message = td\_execute($json\_request)

    Synchronously executes a TDLib request. A request can be executed synchronously, only if it is documented with "Can be called synchronously".

- td\_start\_log($max\_verbosity\_level=1024, $buffer\_size=1048576)

    Start to keep log messages and prepare a buffer for them. They will be first stored in a buffer. Then use `td_poll_log()` to take them out.

- td\_stop\_log()

    Stop keeping log messages and wipe out the log buffer.

- td\_poll\_log($cb->($verbosity, $message))

    Fetch and clear the buffered log messages.

# EXAMPLES

This is a short example which implemented user authentication and send a text message.

    use strict;
    use warnings;
    use feature qw(say switch);
    no warnings qw(experimental::smartmatch);

    use Telegram::JsonAPI qw(:all);
    use JSON::XS::ByteString qw(encode_json decode_json);

    td_start_log();

    my $client_id = td_create_client_id;

    td_send($client_id, encode_json({'@type' => 'getAuthorizationState', '@extra' => \1.01234}));

    while(1) {
      if( $authed ) {
        my($mn, $hr) = (localtime)[1,2];
        if( $hr==7 && $mn >= 58 || $hr==8 && $mn < 2 ) {
          say "$hr:$mn fetching...";
          if( my $q = fetch_daily_question() ) {
            say encode_json($q);
            $q->{date} =~ s/\D//g;
          }
        } else {
          say "$hr:$mn skiped.";
        }
      }
      td_poll_log sub { say "got log: @_"; };
      my $msg = td_receive(1);
      if( defined $msg ) {
        say "recv: $msg";
        $msg = decode_json($msg);
        given($msg->{'@type'}) {
          when('updateAuthorizationState') {
            given( $msg->{authorization_state}{'@type'} ) {
              when('authorizationStateWaitTdlibParameters') {
                td_send($cid, encode_json({
                  '@type' => 'setTdlibParameters',
                  parameters => {
                    database_directory => 'tdlib', # path for TDLib to store session data
                    use_message_database => 1,
                    use_secret_chats => 1,
                    api_id => $api_id,     # $api_id and $api_hash could be retrieved from
                    api_hash => $api_hash, #   https://my.telegram.org/apps
                    system_language_code => 'en',
                    device_model => 'Desktop',
                    application_version => '1.0',
                    enable_storage_optimizer => 1,
                  },
                }));
              }
              when('authorizationStateWaitEncryptionKey') {
                td_send($cid, encode_json({
                  '@type' => 'checkDatabaseEncryptionKey',
                  encryption_key => '',
                }));
              }
              when('authorizationStateWaitPhoneNumber') {
                say 'Please enter your phone number:';
                my $phone = <STDIN>;
                $phone =~ s/\s//g;
                td_send($cid, encode_json({
                  '@type' => 'setAuthenticationPhoneNumber',
                  phone_number => $phone,
                }));
              }
              when('authorizationStateWaitCode') {
                say 'Please enter the authentication code you received:';
                my $code = <STDIN>;
                $code =~ s/\s//g;
                td_send($cid, encode_json({
                  '@type' => 'checkAuthenticationCode',
                  code => $code,
                }));
              }
              when('authorizationStateWaitRegistration') {
                say 'Please enter your first name:';
                my $first_name = <STDIN>;
                $first_name =~ s/^\s+|\s+$//g;
                say 'Please enter your last name:';
                my $last_name = <STDIN>;
                $last_name =~ s/^\s+|\s+$//g;
                td_send($cid, encode_json({
                  '@type' => 'registerUser',
                  first_name => $first_name,
                  last_name => $last_name,
                }));
              }
              when('authorizationStateWaitPassword') {
                say 'Please enter your password:';
                my $password = <STDIN>;
                chomp $password;
                td_send($cid, encode_json({
                  '@type' => 'checkAuthenticationPassword',
                  password => $password,
                }));
              }
              when('authorizationStateReady') {
                td_send($client_id, encode_json({
                  '@type' => 'sendMessage',
                  chat_id => $chat_id, # beside the chat list, you can also retrive the chat id from any incoming messages
                  input_message_content => {
                    '@type' => 'inputMessageText',
                    text => {
                      '@type' => 'formattedText',
                      text => "Hello, every one.",
                    },
                  },
                }));
              }
            }
          }
        }
      }
    }

# INSTALL

This module needs `libtdjson`. Hopefully your can install it from your OS package manager.
Or you can get it from [https://github.com/tdlib/td](https://github.com/tdlib/td) and build it on your own.

# SEE ALSO

- github

    [https://github.com/CindyLinz/Perl-Telegram-JsonAPI](https://github.com/CindyLinz/Perl-Telegram-JsonAPI)

- Getting started with TDLib

    [https://core.telegram.org/tdlib/getting-started](https://core.telegram.org/tdlib/getting-started)

- TDLib api list

    What to put in the JSON requests and got from the JSON responses.

    [https://core.telegram.org/tdlib/docs/td\_\_api\_8h.html](https://core.telegram.org/tdlib/docs/td__api_8h.html)

# AUTHOR

Cindy Wang (CindyLinz) &lt;cindy@cpan.orgE&lt;/gt>

# COPYRIGHT AND LICENSE

Copyright (C) 2022 by CindyLinz

This library is free software; you can redistribute it and/or modify
it under the same terms as Perl itself, either Perl version 5.30.2 or,
at your option, any later version of Perl 5 you may have available.

# POD ERRORS

Hey! **The above document had some coding errors, which are explained below:**

- Around line 263:

    Unknown E content in E&lt;/gt>
