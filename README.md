Тестовый стенд: X-Forwarded-For через цепочку nginx  
Архитектура 
                              
Docker-сеть proxy_net 

Пользователь :8081 ──► nginx1 ─────────────────────────────► app:8000 │  
Пользователь :8082 ──► nginx_chain ──► nginx2 ──► nginx3 ──► app:8000 │

Ключевая идея решения:
Проблема в том, что proxy\_add\_x\_forwarded\_for (стандартное поведение nginx) просто добавляет IP к уже существующему заголовку, не проверяя его подлинность. Злоумышленник может отправить X-Forwarded-For: 1.2.3.4 и этот IP попадёт в цепочку.  
Решение \- двухуровневый подход:

| Тип nginx | Директива | Эффект |
| :---- | :---- | :---- |
| Edge (принимает от пользователя) |  proxy\_set\_header X-Forwarded-For $remote\_addr |  Перезаписывает весь заголовок, отбрасывая пользовательский  |
| Internal (между nginx)  |  proxy\_set\_header X-Forwarded-For "$http\_x\_forwarded\_for, $remote\_addr" | Добавляет свой IP к накопленной цепочке  |
|  |  |  |

Edge-прокси гарантирует, что цепочка начинается с реального IP клиента.   
Внутренние прокси честно дописывают свои IP.

**Запуск стенда**  
docker-compose up \-d  
Протокол тестирования

**Тест 1: Прямой маршрут (user \=\> nginx1 \=\> app)**  
curl \-s http://localhost:8081/ | python3 \-m json.tool  
Ожидаемый результат:  
{  
  "X-Forwarded-For": "172.18.0.1"  
}  
Один IP — клиентский.

**Тест 2: Полная цепочка (user → nginx\_chain → nginx2 → nginx3 → app)**  
curl \-s http://localhost:8082/ | python3 \-m json.tool  
Ожидаемый результат:  
{  
  "X-Forwarded-For": "172.18.0.1, 172.18.0.X, 172.18.0.Y"  
}  
Три IP: пользователь \+ nginx\_chain \+ nginx2 (nginx3 — последний, он не добавляет себя в заголовок, он его передаёт).

**Тест 3: Попытка подделки \- прямой маршрут**  
curl \-s \-H "X-Forwarded-For: 1.3.3.7" http://localhost:8081/ | python3 \-m json.tool  
**Ожидаемый результат:**  
{  
  "X-Forwarded-For": "172.18.0.1"  
}  
Фейковый IP 1.3.3.7 отброшен. 

**Тест 4: Попытка подделки \- через всю цепочку**  
curl \-s \-H "X-Forwarded-For: 9.9.9.9" http://localhost:8082/ | python3 \-m json.tool  
Ожидаемый результат:  
{  
  "X-Forwarded-For": "172.18.0.1, 172.18.0.X, 172.18.0.Y"  
}  
Фейковый IP 9.9.9.9 отброшен edge-прокси. 

**Тест 5: Проверка через curl verbose**  
\# Показываем все заголовки запроса и ответа  
curl \-sv http://localhost:8082/ 2\>&1 | grep \-E "(X-Forwarded|\< HTTP)"  
Автоматический прогон всех тестов  
chmod \+x test.sh  
./test.sh

**Почему не использовать proxy\_add\_x\_forwarded\_for**  
Эта встроенная переменная nginx эквивалентна:  
proxy\_set\_header X-Forwarded-For "$http\_x\_forwarded\_for, $remote\_addr";  
Она не отбрасывает заголовок от пользователя на edge-прокси.   
Поэтому на edge-прокси её использовать нельзя — только явная перезапись через $remote\_addr.

**Остановка стенда**  
docker-compose down
