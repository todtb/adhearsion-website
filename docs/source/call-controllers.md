# Call Controllers

[TOC]

With the addition of Call Controllers to Adhearsion 2.0, Adhearsion applications become closer to real MVC applications. A CallController is the "controller" in MVC terms; the call object is the "view" (being the method of interaction between the human and the application, it qualifies here, even though it is not actually visible); models may be any backend you prefer, be they backed by a database, a directory (like LDAP) or anything else. Indeed, one might wish to make use of a second view, such as an XMPP interaction or some kind of push-based rendering to a visual display.

So, how does one write an application based on call controllers? Simple: generate a controller class and route calls to it as described in [routing](/docs/routing). That might look something like this:

<pre class="terminal">
{{ d['call_controllers.sh|idio|shint|ansi2html']['generate-controller'] }}
</pre>

config/adhearsion.rb:

<pre class="brush: ruby;">
Adhearsion.routes do
  route 'default', SuperSecretProjectCallController
end
</pre>

lib/super_secret_project_call_controller.rb:

<pre class="brush: ruby;">
{{ a['call_controllers.sh|idio|shint|ansi2html']['generate-controller:files:source/myapp/lib/super_secret_project_call_controller.rb'] }}
</pre>

Here, the controller class itself lives in the lib directory, which, by default, is auto-loaded by Adhearsion. You may configure this like so:

config/adhearsion.rb:

<pre class="brush: ruby;">
Adhearsion.config do |config|
  config.lib = 'application/call_controllers'
end
</pre>

Within a call controller, you have access to many instance methods with which to operate on the call, and you also have access to the call object (#call) itself.

## Basic call control

There are several methods available to exercise basic control over a call. The first interesting one is the method by which you answer a call. Hold on to your hats here, this is highly complex:

<pre class="brush: ruby;">
class SuperSecretProjectCallController < Adhearsion::CallController
  def run
    answer
  end
end
</pre>

Hanging up a call is equally hard work:

<pre class="brush: ruby;">
class SuperSecretProjectCallController < Adhearsion::CallController
  def run
    hangup
  end
end
</pre>

You're beginning to see now why telephony consulting is so profitable!

It is also possible to mute and unmute the call:

<pre class="brush: ruby;">
class SuperSecretProjectCallController < Adhearsion::CallController
  def run
    answer
    mute
    unmute
    hangup
  end
end
</pre>

## Multiple controllers

It is possible to fork call control logic and drop out of the current controller in order to execute another. There are two approaches to this. The first is to invoke another controller *within* the current controller:

<pre class="brush: ruby;">
class SuperSecretProjectCallController < Adhearsion::CallController
  def run
    invoke OtherController
    # After OtherController the call will continue:
    say "Thanks for using OtherController."
  end
end
</pre>

In this case, the new controller is prepared and executed, until it completes, at which point the #invoke method will unblock and control will return to the original controller.

Sometimes, it is desireable to abandon the current controller, and hop to a new one entirely. This is equally as trivial:

<pre class="brush: ruby;">
class SuperSecretProjectCallController < Adhearsion::CallController
  def run
    pass OtherController
    raise "Inconcievable!" # This code is never reached
  end
end
</pre>

## Rendering Output

There are several methods by which to render audible output to the call. The full complement can be found in the [Output API documentation](http://rubydoc.info/github/adhearsion/adhearsion/Adhearsion/CallController/Output), but the main methods are described below:

### #play

\#play allows for rendering output of several types, in a manner appropriate to that type. It will detect Date/Time objects, numbers, and paths/URLs to audio files. Additionally, it supports rendering collections of such objects in series. As always, further details can be found in the [API documentation](http://rubydoc.info/github/adhearsion/adhearsion/Adhearsion/CallController/Output:play), but here are some examples:

<pre class="brush: ruby;">
class MyController < Adhearsion::CallController
  def run
    answer
    play 'file://rub-dub-dub.mp3', 3, 'file://men-in-a-tub.mp3'
    play 'file://the-time-is-now.mp3', Time.now
  end
end
</pre>

### #say

In many circumstances, it is desireable to speak certain output using a text-to-speech engine. This is simple using the #say method, providing either a simple string or an SSML document:

<pre class="brush: ruby;">
class MyController < Adhearsion::CallController
  def run
    answer

    say "Rub dub dub, three men in a tub."

    doc = RubySpeech::SSML.draw do
      string "The time is now"
      say_as interpret_as: 'time' do
        Time.now
      end
    end
    say doc
  end
end
</pre>

As usual, check out the [#say API documentation](http://rubydoc.info/github/adhearsion/adhearsion/Adhearsion/CallController/Output:say) for more.

### Asterisk setup

If a text-to-speech engine is configured, all output will be rendered by the engine; otherwise, only file playback will be supported on Asterisk. The adhearsion-asterisk plugin provides helpers for Asterisk's built in complex-data rendering methods, and [its documentation](http://adhearsion.github.com/adhearsion-asterisk) is the appropriate place to find documentation for those.

#### app_swift & Cepstral

We will not cover setting up app_swift itself here, but configuring Adhearsion to use app_swift is simple:

<pre class="brush: ruby;">
Adhearsion.config do |config|
  config.punchblock.media_engine = :swift
end
</pre>

Once you do this, all audio output, including audio file playback, will be delegated to app_swift. If you wish to combine this with asterisk native playback, you should use the helpers in adhearsion-asterisk.

#### UniMRCP

It is possible to render output documents via an engine attached to Asterisk via MRCP. You should see the [UniMRCP site](http://code.google.com/p/unimrcp/wiki/asteriskUniMRCP) for details on the Asterisk configuration, and again, the Adhearsion configuration is simple:

<pre class="brush: ruby;">
Adhearsion.config do |config|
  config.punchblock.media_engine = :unimrcp
end
</pre>

## Collecting Input

Phone calls are fairly boring if they only involve listening to output, so Adhearsion includes several ways to gather input from the caller.

### #ask

Simple collection of a response from the caller couldn't be easier in Adhearsion. CallController provides the #ask method, which allows combining a question (rendered as above) with gathering a DTMF response.

This is best demonstrated by example:

<pre class="brush: ruby;">
class MyController < Adhearsion::CallController
  def run
    answer
    result = ask "How many woodchucks? Enter a number followed by #.",
                :terminator => '#'
    say "Wow, #{result.response} is a lot of woodchucks!"
  end
end
</pre>

Here, we choose to cease input using a terminator digit. Alternative strategies include a :timeout, or a digit :limit, which are described in the [#ask API documentation](http://rubydoc.info/github/adhearsion/adhearsion/Adhearsion/CallController/Input:ask). Additionally, it is possible to pass a block, to which #ask will yield the digit buffer after each digit is received, in order to validate the input and optionally terminate early. If the block returns a truthy value when invoked, the input will be terminated early.

### #menu

Rapid and painless creation of complex IVRs has always been one of the defining features of Adhearsion for beginning and advanced programmers alike. Through the #menu DSL method, the framework abstracts and packages the output and input management and the complex state machine needed to implement a complete menu with audio prompts, digit checking, retries and failure handling, making creating menus a breeze.

A sample menu might look something like this:

<pre class="brush: ruby;">
class MyController < Adhearsion::CallController
  def run
    answer
    menu "Where can we take you today?",
         :timeout => 8.seconds, :tries => 3 do
      match 1, BooController
      match '2', MyOtherController
      match 3, 4, { pass YetAnotherController }
      match 5, FooController
      match 6..10 do |dialed|
        say_dialed dialed
      end

      timeout { do_this_on_timeout }

      invalid do
        invoke InvalidController
      end

      failure do
        speak 'Goodbye'
        hangup
      end
    end

    speak "This code gets executed unless #pass is used"
  end

  def say_dialed(dialed)
    speak "#{dialed} was dialed"
  end

  def do_this_on_timeout
    speak 'Timeout'
  end
end
</pre>

The first arguments to #menu are a list of sounds to play, as accepted by #play, including strings for TTS, Date and Time objects, and file paths. #play and the other input and output methods, all renovated, will be covered in a subsequent post. Sounds will be played at the beginning of the menu and after each timeout or invalid input, if the maximum number of tries has not been reached yet.

The :tries and :timeout options respectively specify the number of tries before going into failure, and the timeout in seconds allowed before the first and each subsequent digit input.

The most important section is the following block, which specifies how the menu will be constructed and handled.

The #match method takes an Integer, a String, a Range or any number of them as the required input(s) for the match payload to be executed. The last argument to a #match is either the name of a CallController, which will be invoked, or a block to be executed. Matched input is passed in to the associated block, or to the controller through its metadata (Controller#metadata[:extension]).

\#menu executes the payload for the first exact unambiguous match it finds after each input or timing out. In a situation where there might be overlapping patterns, such as 10 and 100, #menu will wait for timeout after the second digit.

\#timeout, #invalid and #failure replace #on_invalid, #on_premature_timeout and #on_failure. These methods only accept blocks as payload, but it is still possible to make use of another CallController by using #pass or #invoke within the block.

\#invalid has its associated block executed when the input does not possibly match any pattern. #timeout block is run when time expires before or between input digits, without there being at least one exact match. #failure runs its block when the maximum number of tries is reached without an input match.

Execution of the current context resumes after #menu finishes. If you wish to jump to an entirely different controller, #pass can be used.

\#menu will return an [CallController::Input::Result](http://rubydoc.info/github/adhearsion/adhearsion/Adhearsion/CallController/Input/Result) object detailing the success or otherwise of the menu, similarly to #ask.

## Recording

NOTE: Currently unsupported on Asterisk

It is easy to record input from a call, using the #record method. There are two approaches to recording calls, satisfying different use-cases.

### Voicemail-like recording

In this case, it is desireable for #record to block until the recording completes, as it is the main feature of that part of the call. This usage is simple:

<pre class="brush: ruby;">
class SuperSecretProjectCall < Adhearsion::CallController
  def run
    recording = record :start_beep => true, :max_duration => 60_000
    logger.info "Recording saved to #{recording.uri}"
    say "Goodbye!"
  end
end
</pre>

### Monitoring recording

Alternatively, it may be desireable to record a call in the background while other interactions occurr, perhaps for training, fact-verification or compliance reasons. In this case, it is necessary to execute the recording asynchronously, and handle its recording as a callback:

<pre class="brush: ruby;">
class SuperSecretProjectCall < Adhearsion::CallController
  def run
    record :async => true do |recording|
      logger.info "Async recording saved to #{recording.uri}"
    end
    say "We are currently recording this call"
    hangup # Triggers the end of the recording
  end
end
</pre>

## Joining calls

If there are multiple calls active in Adhearsion, it is possible to join the media streams of those calls:

<pre class="brush: ruby;">
class SuperSecretProjectCall < Adhearsion::CallController
  def run
    answer
    join some_other_call # or some_other_call_id
    say "We hope you had a nice chat!"
  end
end
</pre>

Here, #join will block until either the 3rd-party call hangs up, or is otherwise unjoined out-of-band. It is possible to do an asynchronous join by specifying :async => true, in which case #join will only block until the join is confirmed.

Calls may be unjoined out-of-band using the Call#unjoin method, which has a similar signature to #join.

## Making outbound calls

Similarly to #join, it is possible to make an outbound call to a third-party, and then to join the calls on answering:

<pre class="brush: ruby;">
class SuperSecretProjectCall < Adhearsion::CallController
  def run
    answer
    dial 'sip:foo@bar.com'
    say "We hope you had a nice chat!"
  end
end
</pre>

It is also possible to make multiple calls in paralell, under a first-to-answer-wins policy:

<pre class="brush: ruby;">
class SuperSecretProjectCall < Adhearsion::CallController
  def run
    answer
    dial 'sip:foo@bar.com', 'tel:+1404....'
    say "We hope you had a nice chat!"
  end
end
</pre>

It is additionally possible to set a timeout on attempting to dial out, and to check the status of the dial after the fact:

<pre class="brush: ruby;">
class SuperSecretProjectCall < Adhearsion::CallController
  def run
    answer
    status = dial 'sip:foo@bar.com', :for => 30.seconds
    case status.result
    when :answer
      say "We hope you had a nice chat!"
    when :error, :timeout
      say "Unfortunately we couldn't get hold of Bob."
    when :no_answer
      say "It looks like Bob didn't want to talk to you."
    end
  end
end
</pre>

By default, the outbound calls will have a caller ID matching that of the party which called in to the original controller. Overriding the caller ID for the outbound call is easy:

<pre class="brush: ruby;">
class SuperSecretProjectCall < Adhearsion::CallController
  def run
    answer
    dial 'sip:foo@bar.com', :from => 'sip:me@here.com'
    say "We hope you had a nice chat!"
  end
end
</pre>

Further details can be found in the [#dial API documentation](http://rubydoc.info/github/adhearsion/adhearsion/Adhearsion/CallController/Dial:dial).

## Callbacks

In addition, it is possible to define some callbacks to be executed at the appropriate time. These are before_call and after_call, they are class methods, and they take either a block or a symbol (called as an instance method) like so:

lib/super_secret_project_call.rb:

<pre class="brush: ruby;">
class SuperSecretProjectCall < Adhearsion::CallController
  before_call do
    @user = User.find_by_phone_number call.from
  end

  after_call :save_user

  def run
    answer
    pin = collect_pin
    ...
  end

  def collect_pin
    input 4
  end

  def save_user
    @user.save
  end
end
</pre>

<div class='docs-progress-nav'>
  <span class='back'>
    Back to <a href="/docs/console">Console</a>
  </span>
  <span class='forward'>
    Continue to <a href="/docs/routing">Routing</a>
  </span>
</div>