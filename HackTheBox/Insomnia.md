#web #server-side #php
Easy челлендж

Надо сделать куку с именем "administrator". Вот главный виновник проблемы
```php
$db = db_connect();
$json_data = request()->getJSON(true);
if (!count($json_data) == 2) {
	return $this->respond("Please provide username and password", 404);
}
$query = $db->table("users")->getWhere($json_data, 1, 0);
$result = $query->getRowArray();
```

`!count($json_data) == 2` дает всегда false, потому что сначала делается "не", а потом сравнение false с 2. Видимо, сравнение между разными типами в php - корректное поведение.

`$db->table("users")->getWhere($json_data, 1, 0);` вот эта конструкция берет json из запроса и делает вот такой запрос:

```sql
SELECT * from users where `key1`=value1 and `key2`=value2 and ... limit 1 offset 0
```

Если передать в качестве пейлоада просто `{"username": "administrator"}`, то нас пустит как админа. Все.