###### Worker Timeout: 

- Khong connect duoc toi db 
- Sai config
- Chay python de debug: entrypoint: [“python”, “-m”, “aoapi”]

###### Cách debug code trong playbook: 

- Dùng hàm orenctl.log(str(…..)) để ghi log trong orenworker

Luồng gen id của (alert, case, ticket): 

- generator_api, redis (compose up -d)

Luồng vào server 220 của team integrations: 

- Bật svp kết nối SOAR API
- Tắt proxy

Luồng gọi từ domain -> xem địa chị gọi url trên portal. 

Đổi string sang json có dấu single quote -> ast.literal_json.
Đổi string sang json đúng format -> json.loads


https://github.com/golang/go/wiki/CommonMistakes

https://golang.org/doc/faq#closures_and_goroutines

once.Do(f())

Golang Intefaces: Define intefaces of type


Git clean code vs garbage code: 

- Clean code: Code of new feature that will be deployed
- Garbage code: Code which is not going to be deployed. 

