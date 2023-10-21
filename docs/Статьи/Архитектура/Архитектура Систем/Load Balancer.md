---
aliases:
  - балансировщик
share: true
---
Балансировщик нагрузки принимает входящий сетевой трафик от клиента и, основываясь на некоторых критериях этого трафика, отправляет эти сообщения на один из нескольких бэкенд-серверов.

![500](attachments/Load-balancer.excalidraw.png)


## Алгоритмы балансировки

- **Round robin**. Соединения распределяются между серверами по очереди.
  Один из способов это сделать - [[Статьи/Архитектура/Архитектура Систем/Round robin DNS|Round robin DNS]]. 
- **Weighted round-robin**. Администратор вручную присваивает разные веса каждому серверу, предполагая, что какие-то сервера могут обработать больше соединений, чем другие. Запросы распределяются по серверам с учетом этих весов.
- **Least connection**. Балансировщик направляет запрос на сервер с наименьшим количеством открытых в данных момент соединений.
Weighted response time: Averages the response time of each server, and combines that with the number of connections each server has open to determine where to send traffic. By sending traffic to the servers with the quickest response time, the algorithm ensures faster service for users.

Resource-based: Distributes load based on what resources each server has available at the time. Specialized software (called an "agent") running on each server measures that server's available CPU and memory, and the load balancer queries the agent before distributing traffic to that server.


IP hash: Combines incoming traffic's source and destination IP addresses and uses a mathematical function to convert it into a hash. Based on the hash, the connection is assigned to a specific server.
Можно использовать для реализации sticky sessions. Когда клиент после соединения всегда перенаправляется к одному и тому же серверу приложений.

## Client-side load-balancing
Pros
Improved latency: direct connection from client to backend server (no proxy extra hops)
Improved scalability: scalability depends only on the number of servers (no bottleneck)
Cons
Complexity has to be handled by the clients
Complex maintenance (libraries update …) as the implementation is language specific
Clients must be trusted


## Sticky sessions

Load balancers mainly use round-robin algorithms for even traffic distribution, but sticky sessions break this method by directing specific clients to the same server, sacrificing load balancing fairness.