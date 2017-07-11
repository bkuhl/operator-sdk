# operator-sdk-php

This is the readme from a project where I served as the lead sdk designer.

* [Installation](#installation)
* [Usage](#usage)
  * [Provisioning Numbers](#provisioning-numbers)
  * [Incoming Calls](#incoming-calls)
  * [Outgoing Calls](#outgoing-calls)
  * [Call Interactions](#call-interactions)
    * [Say](#say)
    * [Play](#play)
    * [Collect](#collect)
    * [Voicemail](#voicemail)
    * [Menu](#menu)
    * [Hangup](#hangup)
* [Laravel/Lumen](#laravel)
  * [Notifications](#laravel-notifications)

<a name="installation"/>

## Installation

_Installation instructions have been removed_

### Lumen
Register the service provider in `bootstrap/app.php`

``` php
$app->register(Operator\LumenServiceProvider::class);
```

<a name="usage"/>

## Usage

```php
$operator = new Operator($apiKey);
```

 > If using Lumen, you can just dependency inject this class.

<a name="provisioning-numbers"/>

### Provisioning Numbers

Provision a phone number in a given country/area:®

```php
$operator->number()
    ->provision()
    ->byAreaCode(555); // returns a PhoneNumber
```

Call configuration can be setup immediately after the number is provisioned:

```php
$operator->number()
    ->provision()
    ->byAreaCode(555)
    ->whenCalled()
    ->say('Congrats on your new digits')
    ->save();
```

 > Currently provisioning specific numbers is not supported.

<a name="incoming-calls"/>

### Incoming Calls

```php
$operator->number('1-555-463-4356')
    ->whenCalled()
    ->say('Thanks for calling')
    ->save();
```

<a name="outgoing-calls"/>

### Outgoing Calls
```php
$operator->number('555-555-5555')
  ->dial('555-555-6666')
  ->say('You just got a call')
  ->hangup()
  ->call();
```

<a name="call-interactions"/>

## Call Interactions

You can interact with a caller or recipient of a call in many ways.  These apply to both inbound and outbound calls.

<a name="say"/>

### Say

Speak text to the other end of the line.  Text will be spoken in a woman's voice.

```php
$operator->number('555-555-5555')
  ->dial('555-555-6666')
  ->say('You just got a call')
  ->hangup()
  ->call();
```

<a name="play"/>

### Play

Play an audio file to the other end of the line.

```php
$operator->number('1-555-463-4356')
    ->whenCalled()
    ->play('https://mysite.com/audio.mp3')
    ->hangup()
    ->save();
```

<a name="collect"/>

### Collect

Prompt the user for input, and save their input in the call session.

Limit input to a specific number of digits.
Specify a URL to use a pre-recorded audio file for the prompt.
```php
$operator->number('555-555-5555')
    ->whenCalled()
    ->collectZipcode(
        'http://mysite.com/zipcodeprompt.mp3',
        5)
    ->save();
```

Finish gathering input when a specific key is pressed.
Use a plain text message string for a text-to-speech prompt.
```php
$operator->number('555-555-5555')
    ->whenCalled()
    ->collectAccountnumber(
        'Please enter your account number, followed by pound.',
        0,
        '#'
    )
    ->save();
```

<a name="voicemail"/>

### Voicemail

When a number is called, prompt the user to leave a voicemail.

```php
$operator->number('555-555-5555')
    ->whenCalled()
    ->recordVoicemail('Thanks for calling XXXX, please leave us a message')
    ->hangup()
    ->save();
```

Voicemail prompts can also be audio files.

```php
->recordVoicemail('https://mysite.com/audio.mp3') // plays the audio file
```

You can also provide a closure, enabling you to configure some advanced options on the fly.

```php
$operator->number('555-555-5555')
    ->whenCalled()
    ->recordVoicemail('Thanks for calling XXXX, please leave us a message', function (Voicemail $voicemail) {
        $voicemail->endRecordingAfterSecondsOfSilence(15);
        $voicemail->transcribe();
    })
    ->hangup()
    ->save();
```

You can also create a `VoicemailBuilder` object that'll allow you to reuse voicemail setups.

```php
use Operator\Telephony\Steps\Voicemail\Builder as VoicemailBuilder;

class MaintenanceInbox extends VoicemailBuilder
{
    private $company;

    public function __construct(Company $company) {
        $this->company = $company;
    }

    /**
     * @return Say|Play
     */
    public function prompt()
    {
        if ($company->hasCustomGreeting() {
            return new Play($company->getGreetingUrl());
        }
        
        return new Say('Please leave your maintenance request');
    }
}
```

```php
$operator->number('555-555-5555')
    ->whenCalled()
    ->recordVoicemail(new MaintenanceInbox($company))
    ->hangup()
    ->save();
```

If there are no constructor dependencies, you can also pass a class path:

```php
$operator->number('555-555-5555')
    ->whenCalled()
    ->recordVoicemail(SalesInbox::class)
    ->hangup()
    ->save();
```

<a name="menu"/>

### Menu

When a number is called, prompt the user with a menu of options

```php
use Operator\Telephony\Steps\Menu;

$operator->number('555-555-5555')
    ->whenCalled()
    ->presentMenu('Here are some brief options', function (Menu $menu) {
        $menu->option(1)->play('http://mysite.com/option1.mp3')->hangup();
        $menu->option(2)->say('You pressed option 2')->hangup();
        $menu->option(3)->presentMenu('Here are a few more options', function (Menu $menu) {
            $menu->option(1)->say('Deeper into menus')->hangup();
        });
    });
    ->save();
```

You can also create a `MenuBuilder` object that'll allow you to reuse Menu setups.

```php
use Operator\Telephony\Steps\Menu;
use Operator\Telephony\Steps\Menu\Builder as MenuBuilder;

class SupportMenu extends MenuBuilder
{
    private $company;

    public function __construct(Company $company) {
        $this->company = $company;
    }

    public function prompt()
    {
        if ($company->hasCustomAudio() {
            return new Play($company->getGreetingUrl());
        }
        
        return new Say('Select 1 to hear and audio recording.  Press 2 to hear spoken text');
    }

    public function options(Menu $menu)
    {
        $menu->option(1)->play('http://mysite.com/option1.mp3')->hangup();
        $menu->option(2)->say('You pressed option 2')->hangup();
    }
}
```

```php
$operator->number('555-555-5555')
    ->whenCalled()
    ->presentMenu(new SupportMenu($company))
    ->hangup()
    ->save();
```

If there are no constructor dependencies, you can also pass a class path:

```php
$operator->number('555-555-5555')
    ->whenCalled()
    ->presentMenu(SupportMenu::class)
    ->hangup()
    ->save();
```

<a name="hangup"/>

### Hangup

End the current call.

```php
$operator->number('1-555-463-4356')
    ->whenCalled()
    ->say('Is your refrigerator running?')
    ->hangup()
    ->save();
```

## Lumen/Laravel Support

The SDK comes out of the box with Laravel/Lumen integration.  To get started, register the service provider:

For Lumen, add this to `bootstrap/app.php`:

```php
$app->register(\Operator\Laravel\ServiceProvider::class);
```

For Laravel, add this to the providers array in `config/app.php`:

```php
\Operator\Laravel\ServiceProvider::class,
```

<a name="laravel-notifications"/>

### Notifications

The SDK will automatically interpret notification callbacks from Operator and fire events within your application.  Supported events are:

 * `Operator\Notifications\CallStateChanged`
 * `Operator\Notifications\NewVoicemail`
 * `Operator\Notifications\Notification` - generic, catch-all for unrecognized notifications (useful if your SDK version gets behind and you're receiving notifications the SDK isn't familiar with)

To handle these callbacks, register event listeners and your classes which handle that event:

```php
class EventServiceProvider
{
    protected $listen = [
        Operator\Notifications\CallStateChanged::class => [
            App\Listeners\LogCallDuration::class,
        ],
    ];
}
```

Your handler should accept that event's class as a parameter:

```php
use Operator\Notifications\CallStateChanged;

class LogCallDuration
{
    public function handle(CallStateChanged $callStateChanged)
    {
        if ($callStateChanged->toCompleted()) {
            // figure out duration and log it
        }
    }
}
```

You'll notice in this case the `CallStateChanged` object provides a helper method for determining what happened.  
