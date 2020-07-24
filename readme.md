# Class *Hotkey*
###### AutoHotkey v1.30.03+ recommended (tested on ahk v1.1.32.00)
***

A simple wrapper for the AutoHotkey Hotkey command.

Known limitations:
> - Hotkeys created using the class behave regardless of [#If-based directives](https://www.autohotkey.com/docs/commands/_IfWinActive.htm) (if any).

Some notes about the implementation:
> - The wrapper internally handles quite naively the 'down' and 'up' events of joystick buttons: by polling when relevant, using both the ``GetKeyState`` (AHK) and the ``SetTimer`` (Win API) functions.
> - The script uses a path indexer to keep track of instances, operate on them and dispose them properly. Instances are indexed using a group name, the IfWin setting (the specifications of the context sensitivity which was active at the time the hotkey instance was created) and the name of the hotkey's activation key.

###### Thanks to ``Runar "RUNIE" Borge`` and ``lord_ne`` whose respective works served as bases for this one.

## Table of Contents
- **[Instances](#instances)**
- **[Properties](#properties)**
- **[Methods](#methods)**
- **[Base object methods](#base-object-methods)**

##

## Instances

</br>

> *An instance of ``Hotkey`` will be from now on referred to as ``hk``.*

Syntax description for creating a new object derived from ``Hotkey``, using the [``new`` keyword](https://autohotkey.com/docs/Objects.htm#Custom_Objects):
```AutoHotkey
hk := new Hotkey(_buttonName, [_enabled:=true])
```

| parameter | description |
|:-|:-|
| ``_buttonName``</br>[*string*] | Name of the hotkey's activation key, including any [modifier symbols](https://www.autohotkey.com/docs/Hotkeys.htm#Symbols) (*e.g.* `!i` for ALT+I, `^u` for CTRL+U, `#v` for WIN+V, ``1Joy3``, ``2Joy7``, ``Joy4`` *etc*.). |
| ``_enabled``</br>*OPTIONAL*</br>[*boolean*] | A boolean which determine whether or not the hotkey should start off in an initially-enabled state (defaults to `true`). |

###### *If the [hotkey variant](https://www.autohotkey.com/docs/commands/_IfWinActive.htm#variant) already exists, the instance being replaced is automatically freed (circular and bound references) before being actually replaced.*

#### example

```AutoHotkey
; new Hotkey("Joy3")
new Hotkey("1Joy3", true) ; same as above
new Hotkey("1Joy3", true).onEvent("f") ; see below for more details
hk := new Hotkey("2Joy4", false)
MsgBox % hk.getKeyName() ; displays '2Joy4'

f(_hk, _state:="") {
/*
if the first parameter - here '_hk' but the name does not matter - is
declared, it is assigned a reference to the hotkey instance each time it
is called as a result of a hotkey press event.
*/
static _i := 0
ToolTip % _hk.getKeyName() "|" _state "|" ++_i, 100, 0, 1
}

```

</br>

## Properties

</br>

> **Important note**: *All properties described below are not publicly implemented: they are, for now, intended for internal use only.*

| [property](https://www.autohotkey.com/docs/Objects.htm#Usage_Objects) | description |
| :---: | :---: |
| ``hk._keypressHandler._ITERATOR_CONSUMMATION_DELAY``</br>[*float*] | The delay, in seconds, after which an up event, after the key had been first held down, will be simulated (and before a down event is then simulated repetitively, as long as the joystick button is hold down). Defaults to ``0.35``. |
| ``hk._keypressHandler._ITERATOR_PERIOD``</br>[*integer*] | While the joystick button is held down, the number of milliseconds that must pass before a new down event is simulated. Defaults to ``65``. For this very reason, the callback chain passed to the [onEvent](#onevent) method to handle the 'down' event should be written to complete quickly. On a side note, hotkeys are created with the followings [options](https://www.autohotkey.com/docs/commands/Hotkey.htm#Parameters): ``B0 T1``. |

</br>

## Methods

</br>

> *Note: Whenever necessary, a method call internally and automatically reasserts the instance's own IfWin setting, which determines the [variant](https://www.autohotkey.com/docs/commands/_IfWinActive.htm#variant) of the hotkey upon which to operate*:

***
```AutoHotkey
Hotkey.IfWinActive("ahk_class Notepad")
hk := new Hotkey("1Joy3")
Hotkey.IfWinActive()
; ... some code ...
hk.enable()
/*
the instance's own IfWin setting ('IfWinActive'; 'ahk_class Notepad')
is automatically temporarily reasserted inside the enable method.
*/
```
***

### getCriteria

***
```AutoHotkey
Hotkey.setGroup("myGroup")
Hotkey.IfWinActive("ahk_class Notepad")
hk := new Hotkey("1Joy3")
criteria := hk.getCriteria()
MsgBox % criteria.1 "," criteria.2 "," criteria.3
; displays 'myGroup', 'IfWinActive', 'ahk_class Notepad'
```
***

###### Returns an array *[OBJECT]* whose values are respectively the group and the specifications of the context sensitivity which was active at the time the hotkey instance was created and which it inherited.
> note: this *instance* method should be distinguished from the [*base* one](#getcriteria-base-method).

### getKeyName

***
```AutoHotkey
hk.getKeyName()
```
***

###### Returns the - todo: normalized - hotkey name *[STRING]* representing the hotkey.

### isEnabled

***
```AutoHotkey
hk.isEnabled()
```
***

###### Returns `1` (`true`) if the hotkey is currently enabled or `0` (`false`) if it is disabled.

### enable / disable / toggle

***
```AutoHotkey
hk.enable()
hk.disable()
hk.toggle()
```
***

###### Respectively enables, disables or toggles a given hotkey.

### delete

***
```AutoHotkey
hk.delete()
hk := "" ; should call __Delete of all related objects including the instance's one, if implemented.
```
***

###### 'Delete' the given hotkey (though this does not delete the instance itself). In particular, this method differs from the disable one since it releases all objects the key is bound to (circular and bound references). Note: if a hotkey is created while the [hotkey variant](https://www.autohotkey.com/docs/commands/_IfWinActive.htm#variant) already exists, the instance being replaced is automatically freed before actually being replaced. Also, all instances are automatically deleted the moment the script exits. Once the method has been called, the script is assumed not to query, interact with the instance thereafter.

### onEvent

***
```AutoHotkey
hk := hk.onEvent(_callbacks*)
```
***

###### Returns the hotkey instance itself and specify for the given hotkey one or more functions to execute when the hotkey is pressed. Callable objects are exceuted in the order in which they have been submitted. If a callback declares at least one parameter, the first one is assigned a reference to the hotkey instance each time the callback is called as a result of a press of its associated hotkey. Additionally, if the hotkey is a joystick hotkey, the callback can declare a second parameter - it will be assigned a value depending on whether the joystick button has just been pressed (``-1``), is currently hold down (``1``) or is now being released (``0``).

| parameter | description |
|:-|:-|
| ``_callbacks``</br>*VARIADIC* *OPTIONAL*</br>[*func object/boundfunc object/object/string*] | One or more functions to execute when the hotkey fires. Each callback parameter can be either the name of a function, a [function reference](https://www.autohotkey.com/docs/commands/Func.htm), a [boundFunc object](https://www.autohotkey.com/docs/objects/Functor.htm#BoundFunc) or an [user-defined function object](https://www.autohotkey.com/docs/objects/Functor.htm#User-Defined). |

</br>

## Base object methods

</br>

### getCriteria (base method)

***
```AutoHotkey
Hotkey.setGroup("myGroup")
Hotkey.IfWinActive("ahk_class Notepad")
Hotkey.getCriteria()
MsgBox % criteria.1 "," criteria.2 "," criteria.3
; displays 'myGroup', 'IfWinActive', 'ahk_class Notepad'
```
***

###### Returns an array *[OBJECT]* whose values are respectively the group and the specifications of the context sensitivity which are currently active and whose new instances of ``Hotkey`` will inherit by default.
> note: this *base* method should be distinguished from the [*instance* one](#getcriteria).

### InTheEvent/InTheEventNot/IfWinActive/IfWinNotActive/IfWinExist/IfWinNotExist

***
```AutoHotkey
Hotkey.InTheEvent([_param:=""])
Hotkey.InTheEventNot([_param:=""])
Hotkey.IfWinActive([_param:=""])
Hotkey.IfWinNotActive([_param:=""])
Hotkey.IfWinExist([_param:=""])
Hotkey.IfWinNotExist([_param:=""])
```
***

###### Make all subsequently-created hotkeys context sensitive. Each call of one of these base methods is mutually exclusive; only the most recent one will be in effect. To turn off context sensitivity (that is, to make subsequently-created hotkeys work in all windows), omit the ``_param`` parameter. Known limitation: hotkeys created using the class behave regardless of [#If-based directives](https://www.autohotkey.com/docs/commands/_IfWinActive.htm) (if any).

| parameter | description |
|:-|:-|
| ``_param`` | **I.** (**'IfWinXXX'**) *OPTIONAL* [*string*]</br>A [window title](https://www.autohotkey.com/docs/misc/WinTitle.htm) or other criteria identifying the target window.</br>**II.** (**'InTheEvent/InTheEventNot'**) [*func object/boundfunc object/object/string*]</br>A single variable or an expression containing the name of a function or an object with a call method. The function or call method can accept one parameter, the name of the hotkey (see also: [If, % FunctionObject](https://www.autohotkey.com/docs/commands/Hotkey.htm#IfFn)). |


### deleteAll / enableAll / disableAll

***
```AutoHotkey
Hotkey.deleteAll(_group:="")
Hotkey.enableAll(_group:="")
Hotkey.disableAll(_group:="")
```
***

###### Respectively deletes, enables or disables a specific group of/all hotkeys.

| parameter | description |
|:-|:-|
| ``_group``</br>*OPTIONAL*</br>[*string*] | The name of a group that was previously set using the [setGroup](#setgroup) base method. If omitted the method operates on all hotkeys created so far by the class. |

>  The method returns the number of hotkey instances actually deleted/enabled/disabled. In cases where one or more instances could not be deleted/enabled/disabled, the method sets ``ErrorLevel`` accordingly.

### setGroup

***
```AutoHotkey
Hotkey.setGroup(_group:="")
```
***

###### Bind together a collection of related hotkeys to operate upon them specifically later, either using the ``deleteAll``, the ``enableAll`` or the ``disableAll`` method.

| parameter | description |
|:-|:-|
| ``_group``</br>*OPTIONAL*</br>[*string*] | The name of the group, which may consist of alphanumeric characters, underscore and non-ASCII characters. |

>  If the first parameter is omitted, the method returns the setting which is currently in effect.