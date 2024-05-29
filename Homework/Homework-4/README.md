# Домашнее задание 4 (Построение Underlay-сети на базе eBGP)
## Содержание
<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- Содержание
   * [Цель домашней работы](#-)
   * [Задача](#)
   * [Топология](#-1)
   * [Введение](#-2)
   * [IP-план](#ip-)
      + [Loopbacks и ASN](#loopbacks-asn)
   * [План работы](#--1)
      + [Шаги для выполнения работы](#--2)
   * [Выполнение работы](#--3)
      + [Настройка Spine-1](#-spine-1)
         - [Базовые настройки](#--4)
         - [Настройка BGP](#-bgp)
         - [Общая распечатка примененной конфигурации](#--5)
      + [Настройка Spine-2](#-spine-2)
      + [Настройка Leaf-1](#-leaf-1)
      + [Настройка Leaf-2](#-leaf-2)
      + [Настройка Leaf-3](#-leaf-3)
   * [Верификация](#-3)
      + [Верификация Spine-1](#-spine-1-1)
         - [Интерфейсы](#-4)
         - [Route-map и Filter-list](#route-map-filter-list)
         - [BGP instance](#bgp-instance)
         - [BGP summary](#bgp-summary)
         - [BGP Neighbors (detail)](#bgp-neighbors-detail)
         - [BGP RIB](#bgp-rib)
         - [BFD peers](#bfd-peers)
      + [Верификация Spine-2](#-spine-2-1)
         - [Интерфейсы](#-5)
         - [Route-map и Filter-list](#route-map-filter-list-1)
         - [BGP instance](#bgp-instance-1)
         - [BGP summary](#bgp-summary-1)
         - [BGP Neighbors (detail)](#bgp-neighbors-detail-1)
         - [BGP RIB](#bgp-rib-1)
         - [BFD peers](#bfd-peers-1)
      + [Верификация Leaf-1](#-leaf-1-1)
         - [Интерфейсы](#-6)
         - [Route-map](#route-map)
         - [BGP instance](#bgp-instance-2)
         - [BGP summary](#bgp-summary-2)
         - [BGP Neighbors (detail)](#bgp-neighbors-detail-2)
         - [BGP RIB](#bgp-rib-2)
         - [BFD peers](#bfd-peers-2)
      + [Верификация Leaf-2](#-leaf-2-1)
         - [Интерфейсы](#-7)
         - [Route-map](#route-map-1)
         - [BGP instance](#bgp-instance-3)
         - [BGP summary](#bgp-summary-3)
         - [BGP Neighbors (detail)](#bgp-neighbors-detail-3)
         - [BGP RIB](#bgp-rib-3)
         - [BFD peers](#bfd-peers-3)
      + [Верификация Leaf-3](#-leaf-3-1)
         - [Интерфейсы](#-8)
         - [Route-map](#route-map-2)
         - [BGP instance](#bgp-instance-4)
         - [BGP summary](#bgp-summary-4)
         - [BGP Neighbors (detail)](#bgp-neighbors-detail-4)
         - [BGP RIB](#bgp-rib-4)
         - [BFD peers](#bfd-peers-4)
      + [Верификация связности между Loopback'ами](#-loopback)
         - [Spine-1](#spine-1)
         - [Spine-2](#spine-2)
         - [Leaf-1](#leaf-1)
         - [Leaf-2](#leaf-2)
         - [Leaf-3](#leaf-3)
      + [Итого](#itogo)

<!-- TOC end -->

<!-- TOC --><a name="-"></a>
## Цель домашней работы
Построение underlay-сети для ЦОД с использованием протокола eBGP (Unnumbered) в качестве протокола динамической маршрутизации.

<!-- TOC --><a name=""></a>
## Задача
Обеспечить связность между Loopback-адресами для всех Spine- и Leaf-коммутаторов.

<!-- TOC --><a name="-1"></a>
## Топология

![Topology](topology.png)

<!-- TOC --><a name="-2"></a>
## Введение
В данной домашней работе от нас требуется собрать underlay-сеть, используя протокол eBGP. Я решил попробовать сделать конфигурацию, максимально переносимую между устройствами (между Leaf'ами или между Spine'ами). То есть наша цель сегодня будет состоять в том, чтобы конфигурация между устройствами одного назначения различалась настолько минимально, насколько это разумно. Конечно, по большей части это касается настроек протокола BGP. :)

Так как адресация на интерфейсах будет радикально отличаться от той, которую я использовал в предыдущих домашних заданиях, я здесь приведу настройки устройств с нуля.

Я решил подробно расписать настройку одного устройства (Spine-1), а настройки для других сетевых устройств привести в максимально сжатом виде, без лишних пояснений, так как в принципе настройки всех устройств будут максимально похожи друг на друга. Дополнительные пояснения я буду приводить только там, где это явно потребуется.

Одна из самых интересных "фишек" нашей настройки будет использование схемы, описанной в RFC 5549: Advertising IPv4 Network Layer Reachability Information with an IPv6 Next Hop, или анонсирование IPv4-сетей с использованием некст-хопа IPv6 по-русски.
Что здесь имеется в виду?

Смысл такой схемы настройки состоит в том, чтобы уйти от адресации BGP-соседей с использованием статически заданных стыковочных сетей. По сути мы можем просто указать BGP на конкретный интерфейс и сказать - пирься со всеми, с кем получится (но в рамках строго определенных критериев). :) А так, как локальные IPv6-адреса автоматически настраиваются протоколом IPv6 на всех интерфейсах, где этот самый IPv6 включен, наши BGP-спикеры найдут друг друга и установят соседство.
При этом никакой IPv4-адресации на стыковочных интерфейсах у нас не будет вообще. IPv4-сети (в данном случае лупбэки) будут анонсироваться с link-local IPv6-некстхопами.

Конечно, мы настроим аутентификацию для BGP. Использовать будем самый обычный MD5-алгоритм. Аутентификация в доверенной сети ЦОД нам нужна не для того, чтобы помешать какому-либо злоумышленнику порушить нашу сеть, а скорее для защиты от BGP-Speaker'а, случайно оказавшегося в стыковочном широковещательном домене и запирившегося с нашими коммутаторами. Что там может прилететь от него и как поведет себя сеть - совершенно неясно, поэтому лучше перестраховаться от таких ситуаций и настроить хотя бы минимальную аутентификацию.

Конечная наша цель состоит в том, чтобы обеспечить связность между всеми Loopback'ами, не используя при этом IPv4-адресацию ни на каких линках вообще!

⚠️ Важное замечание! Достижимости между лупбэками Spine по умолчанию не будет, так как они не будут устанавливать анонсы Loopback друг от друга из-за того, что в AS-PATH будет присутствовать собственная AS. Это механизм предотвращения образования петель в eBGP. Данное ограничение можно обойти через использование команды "allowas-in 1" в настройках пир-группы. Однако так, как в нашей сети никакой нужды Spine'ам общаться между собой нет, мы не будем применять эту команду. Если вдруг возникнет специфическая ситуация, при которой нам необходимо будет обеспечить связность для loopback'ов спайнов, мы эту команду всегда можем отдать.

Все наши устройства будут представлять из себя виртуальные Arista vEOS версии 4.29.2F.

Перечислю то, чего мы хотим добиться при выполнении данной работы:
1) Настроить underlay с помощью eBGP
2) Пиринг между устройствами должен быть настроен на IPv6 Link-Local-адресах
3) Должна быть обеспечена аутентификация между BGP-Speaker'ами
4) Для уменьшения времени реагирования на аварии должен быть настроен BFD с таймерами по умолчанию.
5) Таймеры для самого BGP мы немножко подкрутим в сторону уменьшения.
4) По итогу выполненной работы loopback'и всех устройств должны быть достижимы друг для друга.

<!-- TOC --><a name="ip-"></a>
## IP-план

<!-- TOC --><a name="loopbacks-asn"></a>
### Loopbacks и ASN
Так как мы собираемся использовать eBGP в качестве протокола маршрутизации для underlay, мы будем назначать на каждый Leaf свою private AS, а на Spine'ы - одну общую private AS.

| Устройство | Loopback | ASN   |
| ---------- | -------- | ----- |
| Spine-1    | 10.1.0.1 | 65100 |
| Spine-2    | 10.1.0.2 | 65100 |
| Leaf-1     | 10.1.1.1 | 65101 |
| Leaf-2     | 10.1.1.2 | 65102 |
| Leaf-3     | 10.1.1.3 | 65103 | 

AS 65104, 65105, 65106 зарезервируем на будущее (вдруг в нашем поде появятся еще 3 leaf?).

<!-- TOC --><a name="--1"></a>
## План работы

<!-- TOC --><a name="--2"></a>
### Шаги для выполнения работы
1. Выполнить базовую настройку устройств: прописать hostname, перевести интерфейсы в routed-режим, подписать их и т.д.
2. Настроить IPv6-маршрутизацию, которая понадобится нам для схемы BGP Unnumbered.
3. Настроить BGP-процесс. Активировать BFD.
4. Убедиться, что BGP-сессии между физически связанными устройствами установлены и находятся в состоянии Established, выполнить другие возможные проверки, чтобы убедиться, что BGP-процессы на каждом устройстве работают корректно.
5. Убедиться, что BFD-сессии между устройствами находятся в рабочем состоянии и привязаны к процессу BGP.
6. Убедиться в том, что маршруты до каждого Loopback на всех устройствах были установлены в FIB, и что все Loopback-адреса доступны со всех устройств (ping).


<!-- TOC --><a name="--3"></a>
## Выполнение работы
Как уже говорил в введении, я подробно опишу конфигурацию только для Spine-1. Все остальные коммутаторы будут настроены максимально похоже и для них будут приведены лишь распечатки нужных команд.

<!-- TOC --><a name="-spine-1"></a>
### Настройка Spine-1
<!-- TOC --><a name="--4"></a>
#### Базовые настройки
В первую очередь, перед тем, как начинать что-либо настраивать, мы переключимся на новый маршрутизационный движок ArBGP или, как его еще называют, режим "multi-agent". Без этого режима схему BGP Unnumbered построить на удастся. Точнее, BGP настроить получится, и мы даже получим маршруты в FIB, но фактически трафик с использованием ipv6-nexthop ходить не будет. Перво-наперво мы включаем этот движок потому, что после отдачи данной команды потребуется перезагрузить устройство.
Активировать ArBGP можно с помощью команды `service routing protocols model multi-agent`, отданной в режиме глобальной конфигурации:
```
localhost(config)#service routing protocols model multi-agent
! Change will take effect only after switch reboot
```

Перезагрузились, теперь потихоньку начнем настраивать наш Spine:

Установим hostname:
```
localhost(config)#hostname Spine-1
Spine-1(config)#
```

Включим IPv4 и IPv6 маршрутизацию. IPv6-маршрутизация нам необходима для BGP Unnumbered. Также включаем маршрутизацию IPv4 через IPv6-интерфейсы (это потребуется для BGP Unnumbered).
```
Spine-1(config)#ip routing ipv6 interfaces
Spine-1(config)#ipv6 unicast-routing 
```

Настроим Loopback-интерфейс. Пропишем на нем IP-адрес.
```
Spine-1(config)#interface loopback 0
Spine-1(config-if-Lo0)#ip address 10.1.0.1/32
```

Настроим интерфейсы, смотрящие в сторону Leaf'ов. Здесь мы переводим их в routed-режим, пишем осмысленное описание и включаем IPv6 на интерфейсе (нужно для генерации Link-local адресов).
```
Spine-1(config)#interface ethernet 1
Spine-1(config-if-Et1)#description Link_to_Leaf-1
Spine-1(config-if-Et1)#no switchport
Spine-1(config-if-Et1)#ipv6 enable

Spine-1(config)#interface ethernet 2
Spine-1(config-if-Et2)#description Link_to_Leaf-2
Spine-1(config-if-Et2)#no switchport
Spine-1(config-if-Et2)#ipv6 enable

Spine-1(config)#interface ethernet 3
Spine-1(config-if-Et2)#description Link_to_Leaf-3
Spine-1(config-if-Et2)#no switchport
Spine-1(config-if-Et2)#ipv6 enable
```

Остальные доступные интерфейсы отключим от греха подальше. Когда они нам понадобятся, включить будет не проблема.
```
Spine-1(config)#interface ethernet 4-7
Spine-1(config-if-Et4-7)#shutdown
```

На этом основную настройку можно считать завершенной. Переходим к конфигурации underlay-протокола eBGP.

<!-- TOC --><a name="-bgp"></a>
#### Настройка BGP
Первым шагом, еще до создания роутингового процесса, давайте создадим маршрутную карту и фильтрационный список AS.

Маршрутная карта, которую мы здесь создадим, позволит нам отфильтровать интерфейсы, сети которых мы будем редистрибьютить в BGP. Ведь нам необходимо анонсировать в BGP только адрес Loopback, а всё остальное нужно отфильтровать. Данной маршрутной картой мы отловим только интерфейс Loopback0. Все остальное в нее попадать не будет.
```
route-map BGP_REDISTRIBUTE_CONNECTED permit 10
match interface Loopback0
```

Данный фильтрационный список позволит нам задать диапазон AS. Его мы будем использовать при настройке Leaf'ов BGP-соседей для того, чтобы не указывать AS для каждого Leaf'а вручную. Более того, мы здесь оставим запас из трех AS для будущих Leaf'ов, которых у нас сейчас нет. Таким образом при подключении новых Leaf не придется для них настраивать пиринг BGP отдельно. Данный список разрешает диапазон AS 65001-65006, все остальное в него не попадает.
```
peer-filter LEAFS
10 match as-range 65001-65006 result accept
```

Теперь перейдем к настройке непосредственно BGP. Создадим роутинговый процесс для автономной системы 65100 (AS наших Spine'ов).
```
Spine-1(config)#router bgp 65100
```

Включим ECMP для BGP. Возьмем число с запасом. Всего до 64 путей может быть использовано для ECMP в BGP.
```
Spine-1(config-router-bgp)#maximum-paths 64
```

Создадим пир-группу UNDERLAY. Она будет содержать все BGP-настройки, которые одинаковы для наших соседей. Таким образом, не придется набивать одинаковые настройки для каждого соседа по отдельности.
```
Spine-1(config-router-bgp)#neighbor UNDERLAY peer group
```

Отключим задержку перед уведомлением BGP-соседа об изменениях в нашей таблице маршрутизации. В нашей маленькой закрытой ЦОД-сети, она не так актуальна, как в Большом Интернете.
```
Spine-1(config-router-bgp)#neighbor UNDERLAY out-delay 0
```

Активируем для соседа BFD
```
Spine-1(config-router-bgp)#neighbor UNDERLAY bfd
```

Подкрутим Keepalive и Hold таймеры в сторону уменьшения. Это позволит уменьшить время сходимости.
```
Spine-1(config-router-bgp)#neighbor UNDERLAY timers 3 9
```

Включим аутентификацию с помощью MD5-алгоритма. Это позволит защититься от теоретически внезапно могущих появиться "левых" BGP-спикеров на наших стыковочных каналах.
```
Spine-1(config-router-bgp)#neighbor UNDERLAY password OTUS
```

Включим редистрибьюцию IPv4-маршрутов в наш BGP. Тут мы применяем ту самую маршрутную карту, которую мы создали ранее. Это позволит нам анонсировать в BGP исключительно сети интерфейса Loopback0
```
Spine-1(config-router-bgp)#redistribute connected route-map BGP_REDISTRIBUTE_CONNECTED
```

Объявляем наших "нейборов". А точнее, указываем интерфейсы, на которых мы хотим пириться с соседями. В нашем случае это Ethernet1-3, на которых расположены наши Leaf'ы, а также интерфейсы Ethernet 3-6, на которых в будущем могут разместиться новые Leaf-коммутаторы. :)
Также применяем наш фильтрационный список AS, согласно которому пириться мы будем только с теми BGP-Speaker'ами, чьи AS совпадают с нашим списком (65001-65006).
Соседство будет установлено с помощью IPv6 Link-Local адресов.
```
Spine-1(config-router-bgp)#neighbor interface Et1-6 peer-group UNDERLAY peer-filter LEAFS
```

Активируем в BGP семейство адресов IPv4 Unicast
```
Spine-1(config-router-bgp)#address-family ipv4
```

Активируем семейство адресов IPv4 Unicast для нашей группы UNDERLAY и указываем, что в качестве next-hop'а для наших анонсов будут использоваться link-local адреса IPv6.
```
Spine-1(config-router-bgp-af)#neighbor UNDERLAY activate
Spine-1(config-router-bgp-af)#neighbor UNDERLAY next-hop address-family ipv6 originate
```

<!-- TOC --><a name="--5"></a>
#### Общая распечатка примененной конфигурации
На этом настройку нашего Spine-1 можно считать законченной. Приведем полную распечатку команд, которые мы вводили, для наглядности (и для более удобного копипастинга в Spine-2) :)
```
service routing protocols model multi-agent
!
hostname Spine-1
!
interface Ethernet1
   description Link_to_Leaf-1
   no switchport
   ipv6 enable
!
interface Ethernet2
   description Link_to_Leaf-2
   no switchport
   ipv6 enable
!
interface Ethernet3
   description Link_to_Leaf-3
   no switchport
   ipv6 enable
!
interface Ethernet4
   shutdown
!
interface Ethernet5
   shutdown
!
interface Ethernet6
   shutdown
!
interface Ethernet7
   shutdown
!
interface Loopback0
   ip address 10.1.0.1/32
!
ip routing ipv6 interfaces
!
ipv6 unicast-routing
!
route-map BGP_REDISTRIBUTE_CONNECTED permit 10
   match interface Loopback0
!
peer-filter LEAFS
   10 match as-range 65001-65006 result accept
!
router bgp 65100
   maximum-paths 64
   neighbor UNDERLAY peer group
   neighbor UNDERLAY out-delay 0
   neighbor UNDERLAY bfd
   neighbor UNDERLAY timers 3 9
   neighbor UNDERLAY password OTUS
   redistribute connected route-map BGP_REDISTRIBUTE_CONNECTED
   neighbor interface Et1-6 peer-group UNDERLAY peer-filter LEAFS
   !
   address-family ipv4
      neighbor UNDERLAY activate
      neighbor UNDERLAY next-hop address-family ipv6 originate
!
end
```

<!-- TOC --><a name="-spine-2"></a>
### Настройка Spine-2
Здесь всё так же, как и для Spine-1. Меняется только Loopback и hostname.

```
service routing protocols model multi-agent
!
hostname Spine-2
!
interface Ethernet1
   description Link_to_Leaf-1
   no switchport
   ipv6 enable
!
interface Ethernet2
   description Link_to_Leaf-2
   no switchport
   ipv6 enable
!
interface Ethernet3
   description Link_to_Leaf-3
   no switchport
   ipv6 enable
!
interface Ethernet4
   shutdown
!
interface Ethernet5
   shutdown
!
interface Ethernet6
   shutdown
!
interface Ethernet7
   shutdown
!
interface Loopback0
   ip address 10.1.0.2/32
!
ip routing ipv6 interfaces
!
ipv6 unicast-routing
!
route-map BGP_REDISTRIBUTE_CONNECTED permit 10
   match interface Loopback0
!
peer-filter LEAFS
   10 match as-range 65001-65006 result accept
!
router bgp 65100
   maximum-paths 64
   neighbor UNDERLAY peer group
   neighbor UNDERLAY out-delay 0
   neighbor UNDERLAY bfd
   neighbor UNDERLAY timers 3 9
   neighbor UNDERLAY password OTUS
   redistribute connected route-map BGP_REDISTRIBUTE_CONNECTED
   neighbor interface Et1-6 peer-group UNDERLAY peer-filter LEAFS
   !
   address-family ipv4
      neighbor UNDERLAY activate
      neighbor UNDERLAY next-hop address-family ipv6 originate
!
end
```

<!-- TOC --><a name="-leaf-1"></a>
### Настройка Leaf-1
Здесь есть небольшие различия от настройки Spine-ов. Во-первых, так как AS Spine'ов у нас одна, то и фильтрационный список мы указывать не будем, а явно зададим "remote-as 65100". Во-вторых локальная AS для каждого Leaf у нас уникальна. В остальном все то же самое, если не считать мелочи, вроде описаний интерфейсов и количества аплинков.

```
service routing protocols model multi-agent
!
hostname Leaf-1
!
interface Ethernet1
   description Link_to_Spine-1
   no switchport
   ipv6 enable
!
interface Ethernet2
   description Link_to_Spine-2
   no switchport
   ipv6 enable
!
interface Ethernet3
   shutdown
!
interface Ethernet4
   shutdown
!
interface Ethernet5
   shutdown
!
interface Ethernet6
   shutdown
!
interface Ethernet7
   shutdown
!
interface Loopback0
   ip address 10.1.1.1/32
!
ip routing ipv6 interfaces
!
ipv6 unicast-routing
!
route-map BGP_REDISTRIBUTE_CONNECTED permit 10
   match interface Loopback0
!
router bgp 65001
   maximum-paths 64
   neighbor UNDERLAY peer group
   neighbor UNDERLAY out-delay 0
   neighbor UNDERLAY bfd
   neighbor UNDERLAY timers 3 9
   neighbor UNDERLAY password OTUS
   redistribute connected route-map BGP_REDISTRIBUTE_CONNECTED
   neighbor interface Et1-2 peer-group UNDERLAY remote-as 65100
   !
   address-family ipv4
      neighbor UNDERLAY activate
      neighbor UNDERLAY next-hop address-family ipv6 originate
!
end
```

<!-- TOC --><a name="-leaf-2"></a>
### Настройка Leaf-2
Всё то же самое, что и для Leaf-1. Другой IP на Loopback, другая локальная AS (65002).

```
service routing protocols model multi-agent
!
hostname Leaf-2
!
interface Ethernet1
   description Link_to_Spine-1
   no switchport
   ipv6 enable
!
interface Ethernet2
   description Link_to_Spine-2
   no switchport
   ipv6 enable
!
interface Ethernet3
   shutdown
!
interface Ethernet4
   shutdown
!
interface Ethernet5
   shutdown
!
interface Ethernet6
   shutdown
!
interface Ethernet7
   shutdown
!
interface Loopback0
   ip address 10.1.1.2/32
!
ip routing ipv6 interfaces
!
ipv6 unicast-routing
!
route-map BGP_REDISTRIBUTE_CONNECTED permit 10
   match interface Loopback0
!
router bgp 65002
   maximum-paths 64
   neighbor UNDERLAY peer group
   neighbor UNDERLAY out-delay 0
   neighbor UNDERLAY bfd
   neighbor UNDERLAY timers 3 9
   neighbor UNDERLAY password OTUS
   redistribute connected route-map BGP_REDISTRIBUTE_CONNECTED
   neighbor interface Et1-2 peer-group UNDERLAY remote-as 65100
   !
   address-family ipv4
      neighbor UNDERLAY activate
      neighbor UNDERLAY next-hop address-family ipv6 originate
!
end
```

<!-- TOC --><a name="-leaf-3"></a>
### Настройка Leaf-3
Всё то же самое, что и для Leaf-1 и Leaf-2. Другой IP на Loopback, другая локальная AS (65003).

```
service routing protocols model multi-agent
!
hostname Leaf-3
!
interface Ethernet1
   description Link_to_Spine-1
   no switchport
   ipv6 enable
!
interface Ethernet2
   description Link_to_Spine-2
   no switchport
   ipv6 enable
!
interface Ethernet3
   shutdown
!
interface Ethernet4
   shutdown
!
interface Ethernet5
   shutdown
!
interface Ethernet6
   shutdown
!
interface Ethernet7
   shutdown
!
interface Loopback0
   ip address 10.1.1.3/32
!
ip routing ipv6 interfaces
!
ipv6 unicast-routing
!
route-map BGP_REDISTRIBUTE_CONNECTED permit 10
   match interface Loopback0
!
router bgp 65003
   maximum-paths 64
   neighbor UNDERLAY peer group
   neighbor UNDERLAY out-delay 0
   neighbor UNDERLAY bfd
   neighbor UNDERLAY timers 3 9
   neighbor UNDERLAY password OTUS
   redistribute connected route-map BGP_REDISTRIBUTE_CONNECTED
   neighbor interface Et1-2 peer-group UNDERLAY remote-as 65100
   !
   address-family ipv4
      neighbor UNDERLAY activate
      neighbor UNDERLAY next-hop address-family ipv6 originate
!
end
```

<!-- TOC --><a name="-3"></a>
## Верификация

<!-- TOC --><a name="-spine-1-1"></a>
### Верификация Spine-1
<!-- TOC --><a name="-4"></a>
#### Интерфейсы
Проверим IPv4 и IPv6 адресации

```
Spine-1#show ip interface brief
                                                                        Address
Interface        IP Address       Status      Protocol           MTU    Owner  
---------------- ---------------- ----------- ------------- ----------- -------
Ethernet1        unassigned       up          up                1500           
Ethernet2        unassigned       up          up                1500           
Ethernet3        unassigned       up          up                1500           
Loopback0        10.1.0.1/32      up          up               65535           
Management1      unassigned       down        down              1500           
```

```
Spine-1#show ipv6 interface brief
Interface  Status   MTU  IPv6 Address                   Addr State  Addr Source
---------- ------- ----- ----------------------------- ------------ -----------
Et1        up      1500  fe80::5284:b8ff:fe48:4dcb/64   up          link local 
Et2        up      1500  fe80::5284:b8ff:fe48:4dcb/64   up          link local 
Et3        up      1500  fe80::5284:b8ff:fe48:4dcb/64   up          link local 
```

Все выглядит в порядке. Link-local адреса на месте, IPv4-loopback присутствует

<!-- TOC --><a name="route-map-filter-list"></a>
#### Route-map и Filter-list
Проверим, что список ASN и маршрутная карта созданы корректно.

```
Spine-1#show peer-filter LEAFS
peer-filter LEAFS
      10 match as-range 65001-65006 result accept
```
```     
Spine-1#show route-map BGP_REDISTRIBUTE_CONNECTED
route-map BGP_REDISTRIBUTE_CONNECTED permit 10
  Description:
  Match clauses:
    match interface Loopback0
  SubRouteMap:
  Set clauses:
```
<!-- TOC --><a name="bgp-instance"></a>
#### BGP instance
Здесь можно посмотреть общую информацию, относящуюся к нашему инстансу BGP. Все выглядит корректно. Видим, что на редистрибьюцию connected-маршрутов в IPv4 Unicast применяется наша маршрутная карта.

```
Spine-1#show bgp instance
BGP instance information for VRF default
BGP Local AS: 65100, Router ID: 10.1.0.1
Total peers:             3
  Static peers:          3
  Dynamic peers:         0
  Disabled peers:        0
  Established peers:     3
Four Octet ASN mode enabled
Graceful restart helper mode enabled
Graceful restart mode disabled
Graceful restart timer timeout: 00:05:00
End of rib timer timeout: 00:05:00
Attributes of the reflected routes are not preserved
Number of Adj-RIB-Ins awaiting cleanup: 0
UCMP mode: disabled
Peer mac resolution timeout: 00:00:00
Next hop labeled unicast origination LFIB uninstall delay timer: 04:00:00
BGP IPv4 Listen Port Status: listening on port 179
BGP IPv6 Listen Port Status: listening on port 179
BGP Convergence information:
    BGP has converged:   yes,   Time taken to converge: 00:01:31
    Outstanding EORs:    0,     Outstanding Keepalives: 0
    Convergence timeout: 00:05:00
BGP Convergence timer is inactive
BGP Convergence slow-peer timeout: 00:01:30
Address family IPv4 Unicast:
  Redistributed routes into BGP:
    Connected, Route map: BGP_REDISTRIBUTE_CONNECTED
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Convergence based update synchronization is disabled
  Extended next-hop capability is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
Address family IPv4 MplsLabel:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
  Target RIBs: Tunnel RIB
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family IPv4 MplsVpn:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family IPv6 Unicast:
  Redistributed routes into BGP:
    Connected, Route map: BGP_REDISTRIBUTE_CONNECTED
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  6PE Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib
  6PE Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
Address family IPv6 MplsLabel:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
  Target RIBs: Tunnel RIB
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family IPv6 MplsVpn:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family L2VPN VPLS:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
Address family L2VPN EVPN:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  VXLAN Resolution RIBs: system-unicast-rib
  VXLAN Default Resolution RIBs: system-unicast-rib
  MPLS Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  MPLS Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
```

<!-- TOC --><a name="bgp-summary"></a>
#### BGP summary
Здесь мы видим, что наши соседства с Leaf-коммутаторами установлены. Номера AS корректные, локальный RouterID берется из Loopback0, состояние сессий Established, AFI/SAFI верный - IPv4 Unicast.
```
Spine-1#show bgp summary
BGP summary information for VRF default
Router identifier 10.1.0.1, local AS number 65100
Neighbor                               AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
----------------------------- ----------- ------------- ----------------------- -------------- ---------- ----------
fe80::52b7:2fff:fec9:d5a9%Et1       65001 Established   IPv4 Unicast            Negotiated              1          1
fe80::52e1:2eff:feb6:c5d%Et2        65002 Established   IPv4 Unicast            Negotiated              1          1
fe80::52fc:7cff:fed4:3ea7%Et3       65003 Established   IPv4 Unicast            Negotiated              1          1
```

<!-- TOC --><a name="bgp-neighbors-detail"></a>
#### BGP Neighbors (detail)
Здесь мы можем убедиться, что на соседей применен корректный peer-filter LEAFS, сосед наследует конфигурацию из пир-группы UNDERLAY, keepalive и hold таймеры составляют 3 и 9 секунд соответственно. BFD включен для каждого соседа и находится в состоянии Up.

```
Spine-1#show bgp neighbors
BGP neighbor is fe80::52b7:2fff:fec9:d5a9%Et1, remote AS 65001, external link
 LLDP Neighbors: Leaf-1 (Arista Networks EOS version 4.29.2F running on an Arista vEOS-lab)
  BGP version 4, remote router ID 10.1.1.1, VRF default
  Inherits configuration from and member of peer-group UNDERLAY
  peer filter LEAFS
  Last read 00:00:01, last write 00:00:01
  Hold time is 9, keepalive interval is 3 seconds
  Configured hold time is 9, keepalive interval is 3 seconds
  Effective minimum hold time is 3 seconds
  Hold timer is active, time left: 00:00:08
  Keepalive timer is active, time left: 00:00:01
  Connect timer is inactive
  Idle-restart timer is inactive
  BGP state is Established, up for 00:03:42
  Number of transitions to established: 1
  Last state was OpenConfirm
  Last event was Established
  Types of communities advertised: none
  Enhanced route refresh stale path removal disabled
  Outbound enhanced route refresh enabled
  Neighbor Capabilities:
    Multiprotocol IPv4 Unicast: advertised and received and negotiated
    Four Octet ASN: advertised and received and negotiated
    Route Refresh: advertised and received and negotiated
    Enhanced route refresh: advertised and received and negotiated
    Send End-of-RIB messages: advertised and received and negotiated
    Additional-paths recv capability:
      IPv4 Unicast: advertised
    Additional-paths send capability:
      IPv4 Unicast: received
    Extended Next-Hop Capability:
      IPv4 Unicast: advertised and received and negotiated
  Restart timer is inactive
  End of rib timer is inactive
    IPv4 Unicast End-of-RIB received: Yes
      Received 00:03:41
      Number of stale paths removed after graceful restart: 0
  AIGP attribute send and receive for IPv4 Unicast are disabled
  AIGP attribute send and receive for IPv4 with MPLS Labels are disabled
  AIGP attribute send and receive for IPv6 Unicast are disabled
  AIGP attribute send and receive for IPv6 with MPLS Labels are disabled
  BGP session driven failover for IPv4 Unicast is disabled
  BGP session driven failover for IPv6 Unicast is disabled
  Message Statistics:
                                  Sent      Rcvd
    Opens:                           1         1
    Notifications:                   0         0
    Updates:                         4         5
    Keepalives:                     88        90
    Enhanced Route Refresh:          0         0
    Begin of Route Refresh:          0         0
    End of Route Refresh:            0         0
    Total messages:                 93        96
  Prefix Statistics:
                                   Sent      Rcvd     Best Paths     Best ECMP Paths
    IPv4 Unicast:                     3         1              1                   0
    IPv6 Unicast:                     0         0              0                   0
  Configured maximum total number of routes is 256000, warning limit is 204800
  Inbound updates dropped by reason:
    AS path loop detection: 3
    Cluster ID loop detection: 0
    Enforced First AS: 0
    Malformed MPBGP routes: 0
    Originator ID matches local router ID: 0
    Nexthop matches local IP address: 0
    Unexpected IPv6 nexthop for IPv4 routes: 0
  Inbound updates with attribute errors:
    Resulting in removal of all paths in update (treat as withdraw): 0
    Resulting in AFI/SAFI disable: 0
    Resulting in attribute ignore: 0
    Disabled AFI/SAFIs: None
  Inbound paths dropped by reason:
    IPv4 unicast NLRIs dropped due to martian prefix: 0
    IPv6 unicast NLRIs dropped due to martian prefix: 0
    IPv4 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv4 labeled-unicast NLRIs dropped due to martian prefix: 0
    IPv6 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv6 labeled-unicast NLRIs dropped due to martian prefix: 0
    VPN-IPv4 NLRIs dropped due to route import match failure: 0
    VPN-IPv6 NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to unsupported route type: 0
    Link-state NLRIs dropped because reception is unsupported: 0
    RT Membership NLRIs dropped due to local origin ASN received from external peer: 0
  Outbound paths dropped by reason:
    IPv4 local address not available: 0
    IPv6 local address not available: 0
Local AS is 65100, local router ID 10.1.0.1
TTL is 1
Local TCP address is fe80::5284:b8ff:fe48:4dcb, local port is 35595
Remote TCP address is fe80::52b7:2fff:fec9:d5a9, remote port is 179
Local next hop for next hop self:
  IPv4 Unicast: missing (cannot send updates with next hop self)
BFD is enabled and state is Up
MD5 authentication is enabled
! Sending extended community not configured, updates will be sent without extended communities or route targets
TCP Socket Information:
  TCP state is ESTABLISHED
  Recv-Q: 0/32768
  Send-Q: 0/46080
  Outgoing Maximum Segment Size (MSS): 1440
  Total Number of TCP retransmissions: 0
  Options:
    Timestamps enabled: no
    Selective Acknowledgments enabled: yes
    Window Scale enabled: yes
    Explicit Congestion Notification (ECN) enabled: no
  Socket Statistics:
    Window Scale (wscale): 7,7
    Retransmission Timeout (rto): 204.0ms
    Round-trip Time (rtt/rtvar): 3.5ms/0.4ms
    Delayed Ack Timeout (ato): 40.0ms
    Congestion Window (cwnd): 10
    TCP Throughput: 32.52 Mbps
    Advertised Recv Window (rcv_space): 14400

BGP neighbor is fe80::52e1:2eff:feb6:c5d%Et2, remote AS 65002, external link
 LLDP Neighbors: Leaf-2 (Arista Networks EOS version 4.29.2F running on an Arista vEOS-lab)
  BGP version 4, remote router ID 10.1.1.2, VRF default
  Inherits configuration from and member of peer-group UNDERLAY
  peer filter LEAFS
  Last read 00:00:01, last write 00:00:02
  Hold time is 9, keepalive interval is 3 seconds
  Configured hold time is 9, keepalive interval is 3 seconds
  Effective minimum hold time is 3 seconds
  Hold timer is active, time left: 00:00:08
  Keepalive timer is active, time left: 00:00:00
  Connect timer is inactive
  Idle-restart timer is inactive
  BGP state is Established, up for 00:03:56
  Number of transitions to established: 1
  Last state was OpenConfirm
  Last event was Established
  Types of communities advertised: none
  Enhanced route refresh stale path removal disabled
  Outbound enhanced route refresh enabled
  Neighbor Capabilities:
    Multiprotocol IPv4 Unicast: advertised and received and negotiated
    Four Octet ASN: advertised and received and negotiated
    Route Refresh: advertised and received and negotiated
    Enhanced route refresh: advertised and received and negotiated
    Send End-of-RIB messages: advertised and received and negotiated
    Additional-paths recv capability:
      IPv4 Unicast: advertised
    Additional-paths send capability:
      IPv4 Unicast: received
    Extended Next-Hop Capability:
      IPv4 Unicast: advertised and received and negotiated
  Restart timer is inactive
  End of rib timer is inactive
    IPv4 Unicast End-of-RIB received: Yes
      Received 00:03:54
      Number of stale paths removed after graceful restart: 0
  AIGP attribute send and receive for IPv4 Unicast are disabled
  AIGP attribute send and receive for IPv4 with MPLS Labels are disabled
  AIGP attribute send and receive for IPv6 Unicast are disabled
  AIGP attribute send and receive for IPv6 with MPLS Labels are disabled
  BGP session driven failover for IPv4 Unicast is disabled
  BGP session driven failover for IPv6 Unicast is disabled
  Message Statistics:
                                  Sent      Rcvd
    Opens:                           1         1
    Notifications:                   0         0
    Updates:                         4         4
    Keepalives:                     92        94
    Enhanced Route Refresh:          0         0
    Begin of Route Refresh:          0         0
    End of Route Refresh:            0         0
    Total messages:                 97        99
  Prefix Statistics:
                                   Sent      Rcvd     Best Paths     Best ECMP Paths
    IPv4 Unicast:                     3         1              1                   0
    IPv6 Unicast:                     0         0              0                   0
  Configured maximum total number of routes is 256000, warning limit is 204800
  Inbound updates dropped by reason:
    AS path loop detection: 2
    Cluster ID loop detection: 0
    Enforced First AS: 0
    Malformed MPBGP routes: 0
    Originator ID matches local router ID: 0
    Nexthop matches local IP address: 0
    Unexpected IPv6 nexthop for IPv4 routes: 0
  Inbound updates with attribute errors:
    Resulting in removal of all paths in update (treat as withdraw): 0
    Resulting in AFI/SAFI disable: 0
    Resulting in attribute ignore: 0
    Disabled AFI/SAFIs: None
  Inbound paths dropped by reason:
    IPv4 unicast NLRIs dropped due to martian prefix: 0
    IPv6 unicast NLRIs dropped due to martian prefix: 0
    IPv4 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv4 labeled-unicast NLRIs dropped due to martian prefix: 0
    IPv6 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv6 labeled-unicast NLRIs dropped due to martian prefix: 0
    VPN-IPv4 NLRIs dropped due to route import match failure: 0
    VPN-IPv6 NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to unsupported route type: 0
    Link-state NLRIs dropped because reception is unsupported: 0
    RT Membership NLRIs dropped due to local origin ASN received from external peer: 0
  Outbound paths dropped by reason:
    IPv4 local address not available: 0
    IPv6 local address not available: 0
Local AS is 65100, local router ID 10.1.0.1
TTL is 1
Local TCP address is fe80::5284:b8ff:fe48:4dcb, local port is 179
Remote TCP address is fe80::52e1:2eff:feb6:c5d, remote port is 37385
Local next hop for next hop self:
  IPv4 Unicast: missing (cannot send updates with next hop self)
BFD is enabled and state is Up
MD5 authentication is enabled
! Sending extended community not configured, updates will be sent without extended communities or route targets
TCP Socket Information:
  TCP state is ESTABLISHED
  Recv-Q: 0/32768
  Send-Q: 0/46080
  Outgoing Maximum Segment Size (MSS): 1420
  Total Number of TCP retransmissions: 0
  Options:
    Timestamps enabled: no
    Selective Acknowledgments enabled: yes
    Window Scale enabled: yes
    Explicit Congestion Notification (ECN) enabled: no
  Socket Statistics:
    Window Scale (wscale): 7,7
    Retransmission Timeout (rto): 204.0ms
    Round-trip Time (rtt/rtvar): 3.4ms/0.3ms
    Delayed Ack Timeout (ato): 40.0ms
    Congestion Window (cwnd): 10
    TCP Throughput: 33.24 Mbps
    Advertised Recv Window (rcv_space): 14400

BGP neighbor is fe80::52fc:7cff:fed4:3ea7%Et3, remote AS 65003, external link
 LLDP Neighbors: Leaf-3 (Arista Networks EOS version 4.29.2F running on an Arista vEOS-lab)
  BGP version 4, remote router ID 10.1.1.3, VRF default
  Inherits configuration from and member of peer-group UNDERLAY
  peer filter LEAFS
  Last read 00:00:02, last write 00:00:02
  Hold time is 9, keepalive interval is 3 seconds
  Configured hold time is 9, keepalive interval is 3 seconds
  Effective minimum hold time is 3 seconds
  Hold timer is active, time left: 00:00:07
  Keepalive timer is inactive
  Connect timer is inactive
  Idle-restart timer is inactive
  BGP state is Established, up for 00:03:48
  Number of transitions to established: 1
  Last state was OpenConfirm
  Last event was Established
  Types of communities advertised: none
  Enhanced route refresh stale path removal disabled
  Outbound enhanced route refresh enabled
  Neighbor Capabilities:
    Multiprotocol IPv4 Unicast: advertised and received and negotiated
    Four Octet ASN: advertised and received and negotiated
    Route Refresh: advertised and received and negotiated
    Enhanced route refresh: advertised and received and negotiated
    Send End-of-RIB messages: advertised and received and negotiated
    Additional-paths recv capability:
      IPv4 Unicast: advertised
    Additional-paths send capability:
      IPv4 Unicast: received
    Extended Next-Hop Capability:
      IPv4 Unicast: advertised and received and negotiated
  Restart timer is inactive
  End of rib timer is inactive
    IPv4 Unicast End-of-RIB received: Yes
      Received 00:03:47
      Number of stale paths removed after graceful restart: 0
  AIGP attribute send and receive for IPv4 Unicast are disabled
  AIGP attribute send and receive for IPv4 with MPLS Labels are disabled
  AIGP attribute send and receive for IPv6 Unicast are disabled
  AIGP attribute send and receive for IPv6 with MPLS Labels are disabled
  BGP session driven failover for IPv4 Unicast is disabled
  BGP session driven failover for IPv6 Unicast is disabled
  Message Statistics:
                                  Sent      Rcvd
    Opens:                           1         1
    Notifications:                   0         0
    Updates:                         4         4
    Keepalives:                     91        88
    Enhanced Route Refresh:          0         0
    Begin of Route Refresh:          0         0
    End of Route Refresh:            0         0
    Total messages:                 96        93
  Prefix Statistics:
                                   Sent      Rcvd     Best Paths     Best ECMP Paths
    IPv4 Unicast:                     3         1              1                   0
    IPv6 Unicast:                     0         0              0                   0
  Configured maximum total number of routes is 256000, warning limit is 204800
  Inbound updates dropped by reason:
    AS path loop detection: 2
    Cluster ID loop detection: 0
    Enforced First AS: 0
    Malformed MPBGP routes: 0
    Originator ID matches local router ID: 0
    Nexthop matches local IP address: 0
    Unexpected IPv6 nexthop for IPv4 routes: 0
  Inbound updates with attribute errors:
    Resulting in removal of all paths in update (treat as withdraw): 0
    Resulting in AFI/SAFI disable: 0
    Resulting in attribute ignore: 0
    Disabled AFI/SAFIs: None
  Inbound paths dropped by reason:
    IPv4 unicast NLRIs dropped due to martian prefix: 0
    IPv6 unicast NLRIs dropped due to martian prefix: 0
    IPv4 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv4 labeled-unicast NLRIs dropped due to martian prefix: 0
    IPv6 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv6 labeled-unicast NLRIs dropped due to martian prefix: 0
    VPN-IPv4 NLRIs dropped due to route import match failure: 0
    VPN-IPv6 NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to unsupported route type: 0
    Link-state NLRIs dropped because reception is unsupported: 0
    RT Membership NLRIs dropped due to local origin ASN received from external peer: 0
  Outbound paths dropped by reason:
    IPv4 local address not available: 0
    IPv6 local address not available: 0
Local AS is 65100, local router ID 10.1.0.1
TTL is 1
Local TCP address is fe80::5284:b8ff:fe48:4dcb, local port is 179
Remote TCP address is fe80::52fc:7cff:fed4:3ea7, remote port is 44115
Local next hop for next hop self:
  IPv4 Unicast: missing (cannot send updates with next hop self)
BFD is enabled and state is Up
MD5 authentication is enabled
! Sending extended community not configured, updates will be sent without extended communities or route targets
TCP Socket Information:
  TCP state is ESTABLISHED
  Recv-Q: 0/32768
  Send-Q: 0/46080
  Outgoing Maximum Segment Size (MSS): 1420
  Total Number of TCP retransmissions: 0
  Options:
    Timestamps enabled: no
    Selective Acknowledgments enabled: yes
    Window Scale enabled: yes
    Explicit Congestion Notification (ECN) enabled: no
  Socket Statistics:
    Window Scale (wscale): 7,7
    Retransmission Timeout (rto): 204.0ms
    Round-trip Time (rtt/rtvar): 3.3ms/0.7ms
    Delayed Ack Timeout (ato): 40.0ms
    Congestion Window (cwnd): 10
    TCP Throughput: 33.92 Mbps
    Advertised Recv Window (rcv_space): 14400
```

<!-- TOC --><a name="bgp-rib"></a>
#### BGP RIB
Видим, что маршруты до нужных loopback'ов доступны. AS-PATH выглядят корректными.

```
Spine-1#show bgp ipv4 unicast
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65100
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.1/32            -                     -       -          -       0       i
 * >      10.1.1.1/32            fe80::52b7:2fff:fec9:d5a9%Et1 0       -          100     0       65001 i
 * >      10.1.1.2/32            fe80::52e1:2eff:feb6:c5d%Et2 0       -          100     0       65002 i
 * >      10.1.1.3/32            fe80::52fc:7cff:fed4:3ea7%Et3 0       -          100     0       65003 i
 ```

 #### Route Table
Маршруты до Loopback'ов Leaf'ов в таблице маршрутизации присутствуют.

 ```
Spine-1#show ip route

VRF: default
Codes: C - connected, S - static, K - kernel, 
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 C        10.1.0.1/32 [0/0]
           via Loopback0, directly connected
 B E      10.1.1.1/32 [200/0]
           via fe80::52b7:2fff:fec9:d5a9, Ethernet1
 B E      10.1.1.2/32 [200/0]
           via fe80::52e1:2eff:feb6:c5d, Ethernet2
 B E      10.1.1.3/32 [200/0]
           via fe80::52fc:7cff:fed4:3ea7, Ethernet3
```

<!-- TOC --><a name="bfd-peers"></a>
#### BFD peers
В последнюю очередь проверим состояние наших BFD-сессий. Вроде бы все выглядит в порядке.

```
Spine-1#show bfd peers
VRF name: default
-----------------
DstAddr                        MyDisc    YourDisc  Interface/Transport    Type 
-------------------------- ----------- ----------- -------------------- -------
fe80::52b7:2fff:fec9:d5a9  3310161648  1408405490        Ethernet1(13)  normal 
fe80::52e1:2eff:feb6:c5d   3140695752   350688201        Ethernet2(14)  normal 
fe80::52fc:7cff:fed4:3ea7  1797929343   711531338        Ethernet3(15)  normal 

           LastUp       LastDown            LastDiag    State
-------------------- -------------- ------------------- -----
   05/29/24 07:24             NA       No Diagnostic       Up
   05/29/24 07:24             NA       No Diagnostic       Up
   05/29/24 07:24             NA       No Diagnostic       Up
```

<!-- TOC --><a name="-spine-2-1"></a>
### Верификация Spine-2
<!-- TOC --><a name="-5"></a>
#### Интерфейсы
Проверим IPv4 и IPv6 адресации

```
Spine-2#show ip interface brief
                                                                        Address
Interface        IP Address       Status      Protocol           MTU    Owner  
---------------- ---------------- ----------- ------------- ----------- -------
Ethernet1        unassigned       up          up                1500           
Ethernet2        unassigned       up          up                1500           
Ethernet3        unassigned       up          up                1500           
Loopback0        10.1.0.2/32      up          up               65535           
Management1      unassigned       down        down              1500           

```

```
Spine-2#show ipv6 interface brief
Interface  Status   MTU   IPv6 Address                  Addr State  Addr Source
---------- ------- ----- ----------------------------- ------------ -----------
Et1        up      1500   fe80::5231:cff:fe05:79b7/64   up          link local 
Et2        up      1500   fe80::5231:cff:fe05:79b7/64   up          link local 
Et3        up      1500   fe80::5231:cff:fe05:79b7/64   up          link local 
```

Все выглядит в порядке. Link-local адреса на месте, IPv4-loopback присутствует

<!-- TOC --><a name="route-map-filter-list-1"></a>
#### Route-map и Filter-list
Проверим, что список ASN и маршрутная карта созданы корректно.

```
Spine-2#show peer-filter LEAFS
peer-filter LEAFS
      10 match as-range 65001-65006 result accept
```
```     
Spine-2#show route-map BGP_REDISTRIBUTE_CONNECTED
route-map BGP_REDISTRIBUTE_CONNECTED permit 10
  Description:
  Match clauses:
    match interface Loopback0
  SubRouteMap:
  Set clauses:
```

<!-- TOC --><a name="bgp-instance-1"></a>
#### BGP instance
Здесь можно посмотреть общую информацию, относящуюся к нашему инстансу BGP. Все выглядит корректно. Видим, что на редистрибьюцию connected-маршрутов в IPv4 Unicast применяется наша маршрутная карта.

```
Spine-2#show bgp instance 
BGP instance information for VRF default
BGP Local AS: 65100, Router ID: 10.1.0.2
Total peers:             3
  Static peers:          3
  Dynamic peers:         0
  Disabled peers:        0
  Established peers:     3
Four Octet ASN mode enabled
Graceful restart helper mode enabled
Graceful restart mode disabled
Graceful restart timer timeout: 00:05:00
End of rib timer timeout: 00:05:00
Attributes of the reflected routes are not preserved
Number of Adj-RIB-Ins awaiting cleanup: 0
UCMP mode: disabled
Peer mac resolution timeout: 00:00:00
Next hop labeled unicast origination LFIB uninstall delay timer: 04:00:00
BGP IPv4 Listen Port Status: listening on port 179
BGP IPv6 Listen Port Status: listening on port 179
BGP Convergence information:
    BGP has converged:   yes,   Time taken to converge: 00:01:31
    Outstanding EORs:    0,     Outstanding Keepalives: 0
    Convergence timeout: 00:05:00
BGP Convergence timer is inactive
BGP Convergence slow-peer timeout: 00:01:30
Address family IPv4 Unicast:
  Redistributed routes into BGP:
    Connected, Route map: BGP_REDISTRIBUTE_CONNECTED
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Convergence based update synchronization is disabled
  Extended next-hop capability is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
Address family IPv4 MplsLabel:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
  Target RIBs: Tunnel RIB
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family IPv4 MplsVpn:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family IPv6 Unicast:
  Redistributed routes into BGP:
    Connected, Route map: BGP_REDISTRIBUTE_CONNECTED
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  6PE Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib
  6PE Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
Address family IPv6 MplsLabel:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
  Target RIBs: Tunnel RIB
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family IPv6 MplsVpn:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family L2VPN VPLS:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
Address family L2VPN EVPN:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  VXLAN Resolution RIBs: system-unicast-rib
  VXLAN Default Resolution RIBs: system-unicast-rib
  MPLS Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  MPLS Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
```

<!-- TOC --><a name="bgp-summary-1"></a>
#### BGP summary
Здесь мы видим, что наши соседства с Leaf-коммутаторами установлены. Номера AS корректные, локальный RouterID берется из Loopback0, состояние сессий Established, AFI/SAFI верный - IPv4 Unicast.
```
Spine-2#show bgp summary
BGP summary information for VRF default
Router identifier 10.1.0.2, local AS number 65100
Neighbor                               AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
----------------------------- ----------- ------------- ----------------------- -------------- ---------- ----------
fe80::52b7:2fff:fec9:d5a9%Et1       65001 Established   IPv4 Unicast            Negotiated              1          1
fe80::52e1:2eff:feb6:c5d%Et2        65002 Established   IPv4 Unicast            Negotiated              1          1
fe80::52fc:7cff:fed4:3ea7%Et3       65003 Established   IPv4 Unicast            Negotiated              1          1
```

<!-- TOC --><a name="bgp-neighbors-detail-1"></a>
#### BGP Neighbors (detail)
Здесь мы можем убедиться, что на соседей применен корректный peer-filter LEAFS, сосед наследует конфигурацию из пир-группы UNDERLAY, keepalive и hold таймеры составляют 3 и 9 секунд соответственно. BFD включен для каждого соседа и находится в состоянии Up.

```
Spine-2#show bgp neighbors
BGP neighbor is fe80::52b7:2fff:fec9:d5a9%Et1, remote AS 65001, external link
 LLDP Neighbors: Leaf-1 (Arista Networks EOS version 4.29.2F running on an Arista vEOS-lab)
  BGP version 4, remote router ID 10.1.1.1, VRF default
  Inherits configuration from and member of peer-group UNDERLAY
  peer filter LEAFS
  Last read 00:00:02, last write 00:00:02
  Hold time is 9, keepalive interval is 3 seconds
  Configured hold time is 9, keepalive interval is 3 seconds
  Effective minimum hold time is 3 seconds
  Hold timer is active, time left: 00:00:07
  Keepalive timer is active, time left: 00:00:00
  Connect timer is inactive
  Idle-restart timer is inactive
  BGP state is Established, up for 00:09:50
  Number of transitions to established: 1
  Last state was OpenConfirm
  Last event was Established
  Types of communities advertised: none
  Enhanced route refresh stale path removal disabled
  Outbound enhanced route refresh enabled
  Neighbor Capabilities:
    Multiprotocol IPv4 Unicast: advertised and received and negotiated
    Four Octet ASN: advertised and received and negotiated
    Route Refresh: advertised and received and negotiated
    Enhanced route refresh: advertised and received and negotiated
    Send End-of-RIB messages: advertised and received and negotiated
    Additional-paths recv capability:
      IPv4 Unicast: advertised
    Additional-paths send capability:
      IPv4 Unicast: received
    Extended Next-Hop Capability:
      IPv4 Unicast: advertised and received and negotiated
  Restart timer is inactive
  End of rib timer is inactive
    IPv4 Unicast End-of-RIB received: Yes
      Received 00:09:50
      Number of stale paths removed after graceful restart: 0
  AIGP attribute send and receive for IPv4 Unicast are disabled
  AIGP attribute send and receive for IPv4 with MPLS Labels are disabled
  AIGP attribute send and receive for IPv6 Unicast are disabled
  AIGP attribute send and receive for IPv6 with MPLS Labels are disabled
  BGP session driven failover for IPv6 Unicast is disabled
  BGP session driven failover for IPv4 Unicast is disabled
  Message Statistics:
                                  Sent      Rcvd
    Opens:                           1         1
    Notifications:                   0         0
    Updates:                         4         3
    Keepalives:                    232       233
    Enhanced Route Refresh:          0         0
    Begin of Route Refresh:          0         0
    End of Route Refresh:            0         0
    Total messages:                237       237
  Prefix Statistics:
                                   Sent      Rcvd     Best Paths     Best ECMP Paths
    IPv4 Unicast:                     3         1              1                   0
    IPv6 Unicast:                     0         0              0                   0
  Configured maximum total number of routes is 256000, warning limit is 204800
  Inbound updates dropped by reason:
    AS path loop detection: 1
    Cluster ID loop detection: 0
    Enforced First AS: 0
    Malformed MPBGP routes: 0
    Originator ID matches local router ID: 0
    Nexthop matches local IP address: 0
    Unexpected IPv6 nexthop for IPv4 routes: 0
  Inbound updates with attribute errors:
    Resulting in removal of all paths in update (treat as withdraw): 0
    Resulting in AFI/SAFI disable: 0
    Resulting in attribute ignore: 0
    Disabled AFI/SAFIs: None
  Inbound paths dropped by reason:
    IPv4 unicast NLRIs dropped due to martian prefix: 0
    IPv6 unicast NLRIs dropped due to martian prefix: 0
    IPv4 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv4 labeled-unicast NLRIs dropped due to martian prefix: 0
    IPv6 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv6 labeled-unicast NLRIs dropped due to martian prefix: 0
    VPN-IPv4 NLRIs dropped due to route import match failure: 0
    VPN-IPv6 NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to unsupported route type: 0
    Link-state NLRIs dropped because reception is unsupported: 0
    RT Membership NLRIs dropped due to local origin ASN received from external peer: 0
  Outbound paths dropped by reason:
    IPv4 local address not available: 0
    IPv6 local address not available: 0
Local AS is 65100, local router ID 10.1.0.2
TTL is 1
Local TCP address is fe80::5231:cff:fe05:79b7, local port is 40569
Remote TCP address is fe80::52b7:2fff:fec9:d5a9, remote port is 179
Local next hop for next hop self:
  IPv4 Unicast: missing (cannot send updates with next hop self)
BFD is enabled and state is Up
MD5 authentication is enabled
! Sending extended community not configured, updates will be sent without extended communities or route targets
TCP Socket Information:
  TCP state is ESTABLISHED
  Recv-Q: 0/32768
  Send-Q: 0/46080
  Outgoing Maximum Segment Size (MSS): 1440
  Total Number of TCP retransmissions: 0
  Options:
    Timestamps enabled: no
    Selective Acknowledgments enabled: yes
    Window Scale enabled: yes
    Explicit Congestion Notification (ECN) enabled: no
  Socket Statistics:
    Window Scale (wscale): 7,7
    Retransmission Timeout (rto): 204.0ms
    Round-trip Time (rtt/rtvar): 3.6ms/0.7ms
    Delayed Ack Timeout (ato): 40.0ms
    Congestion Window (cwnd): 10
    TCP Throughput: 31.98 Mbps
    Advertised Recv Window (rcv_space): 14400

BGP neighbor is fe80::52e1:2eff:feb6:c5d%Et2, remote AS 65002, external link
 LLDP Neighbors: Leaf-2 (Arista Networks EOS version 4.29.2F running on an Arista vEOS-lab)
  BGP version 4, remote router ID 10.1.1.2, VRF default
  Inherits configuration from and member of peer-group UNDERLAY
  peer filter LEAFS
  Last read 00:00:01, last write 00:00:02
  Hold time is 9, keepalive interval is 3 seconds
  Configured hold time is 9, keepalive interval is 3 seconds
  Effective minimum hold time is 3 seconds
  Hold timer is active, time left: 00:00:08
  Keepalive timer is active, time left: 00:00:00
  Connect timer is inactive
  Idle-restart timer is inactive
  BGP state is Established, up for 00:10:02
  Number of transitions to established: 1
  Last state was OpenConfirm
  Last event was Established
  Types of communities advertised: none
  Enhanced route refresh stale path removal disabled
  Outbound enhanced route refresh enabled
  Neighbor Capabilities:
    Multiprotocol IPv4 Unicast: advertised and received and negotiated
    Four Octet ASN: advertised and received and negotiated
    Route Refresh: advertised and received and negotiated
    Enhanced route refresh: advertised and received and negotiated
    Send End-of-RIB messages: advertised and received and negotiated
    Additional-paths recv capability:
      IPv4 Unicast: advertised
    Additional-paths send capability:
      IPv4 Unicast: received
    Extended Next-Hop Capability:
      IPv4 Unicast: advertised and received and negotiated
  Restart timer is inactive
  End of rib timer is inactive
    IPv4 Unicast End-of-RIB received: Yes
      Received 00:10:01
      Number of stale paths removed after graceful restart: 0
  AIGP attribute send and receive for IPv4 Unicast are disabled
  AIGP attribute send and receive for IPv4 with MPLS Labels are disabled
  AIGP attribute send and receive for IPv6 Unicast are disabled
  AIGP attribute send and receive for IPv6 with MPLS Labels are disabled
  BGP session driven failover for IPv6 Unicast is disabled
  BGP session driven failover for IPv4 Unicast is disabled
  Message Statistics:
                                  Sent      Rcvd
    Opens:                           1         1
    Notifications:                   0         0
    Updates:                         4         4
    Keepalives:                    238       239
    Enhanced Route Refresh:          0         0
    Begin of Route Refresh:          0         0
    End of Route Refresh:            0         0
    Total messages:                243       244
  Prefix Statistics:
                                   Sent      Rcvd     Best Paths     Best ECMP Paths
    IPv4 Unicast:                     3         1              1                   0
    IPv6 Unicast:                     0         0              0                   0
  Configured maximum total number of routes is 256000, warning limit is 204800
  Inbound updates dropped by reason:
    AS path loop detection: 2
    Cluster ID loop detection: 0
    Enforced First AS: 0
    Malformed MPBGP routes: 0
    Originator ID matches local router ID: 0
    Nexthop matches local IP address: 0
    Unexpected IPv6 nexthop for IPv4 routes: 0
  Inbound updates with attribute errors:
    Resulting in removal of all paths in update (treat as withdraw): 0
    Resulting in AFI/SAFI disable: 0
    Resulting in attribute ignore: 0
    Disabled AFI/SAFIs: None
  Inbound paths dropped by reason:
    IPv4 unicast NLRIs dropped due to martian prefix: 0
    IPv6 unicast NLRIs dropped due to martian prefix: 0
    IPv4 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv4 labeled-unicast NLRIs dropped due to martian prefix: 0
    IPv6 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv6 labeled-unicast NLRIs dropped due to martian prefix: 0
    VPN-IPv4 NLRIs dropped due to route import match failure: 0
    VPN-IPv6 NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to unsupported route type: 0
    Link-state NLRIs dropped because reception is unsupported: 0
    RT Membership NLRIs dropped due to local origin ASN received from external peer: 0
  Outbound paths dropped by reason:
    IPv4 local address not available: 0
    IPv6 local address not available: 0
Local AS is 65100, local router ID 10.1.0.2
TTL is 1
Local TCP address is fe80::5231:cff:fe05:79b7, local port is 179
Remote TCP address is fe80::52e1:2eff:feb6:c5d, remote port is 40671
Local next hop for next hop self:
  IPv4 Unicast: missing (cannot send updates with next hop self)
BFD is enabled and state is Up
MD5 authentication is enabled
! Sending extended community not configured, updates will be sent without extended communities or route targets
TCP Socket Information:
  TCP state is ESTABLISHED
  Recv-Q: 0/32768
  Send-Q: 0/46080
  Outgoing Maximum Segment Size (MSS): 1420
  Total Number of TCP retransmissions: 0
  Options:
    Timestamps enabled: no
    Selective Acknowledgments enabled: yes
    Window Scale enabled: yes
    Explicit Congestion Notification (ECN) enabled: no
  Socket Statistics:
    Window Scale (wscale): 7,7
    Retransmission Timeout (rto): 204.0ms
    Round-trip Time (rtt/rtvar): 3.5ms/0.6ms
    Delayed Ack Timeout (ato): 40.0ms
    Congestion Window (cwnd): 10
    TCP Throughput: 32.21 Mbps
    Advertised Recv Window (rcv_space): 14400

BGP neighbor is fe80::52fc:7cff:fed4:3ea7%Et3, remote AS 65003, external link
 LLDP Neighbors: Leaf-3 (Arista Networks EOS version 4.29.2F running on an Arista vEOS-lab)
  BGP version 4, remote router ID 10.1.1.3, VRF default
  Inherits configuration from and member of peer-group UNDERLAY
  peer filter LEAFS
  Last read 00:00:03, last write 00:00:01
  Hold time is 9, keepalive interval is 3 seconds
  Configured hold time is 9, keepalive interval is 3 seconds
  Effective minimum hold time is 3 seconds
  Hold timer is active, time left: 00:00:06
  Keepalive timer is active, time left: 00:00:01
  Connect timer is inactive
  Idle-restart timer is inactive
  BGP state is Established, up for 00:09:55
  Number of transitions to established: 1
  Last state was OpenConfirm
  Last event was Established
  Types of communities advertised: none
  Enhanced route refresh stale path removal disabled
  Outbound enhanced route refresh enabled
  Neighbor Capabilities:
    Multiprotocol IPv4 Unicast: advertised and received and negotiated
    Four Octet ASN: advertised and received and negotiated
    Route Refresh: advertised and received and negotiated
    Enhanced route refresh: advertised and received and negotiated
    Send End-of-RIB messages: advertised and received and negotiated
    Additional-paths recv capability:
      IPv4 Unicast: advertised
    Additional-paths send capability:
      IPv4 Unicast: received
    Extended Next-Hop Capability:
      IPv4 Unicast: advertised and received and negotiated
  Restart timer is inactive
  End of rib timer is inactive
    IPv4 Unicast End-of-RIB received: Yes
      Received 00:09:54
      Number of stale paths removed after graceful restart: 0
  AIGP attribute send and receive for IPv4 Unicast are disabled
  AIGP attribute send and receive for IPv4 with MPLS Labels are disabled
  AIGP attribute send and receive for IPv6 Unicast are disabled
  AIGP attribute send and receive for IPv6 with MPLS Labels are disabled
  BGP session driven failover for IPv6 Unicast is disabled
  BGP session driven failover for IPv4 Unicast is disabled
  Message Statistics:
                                  Sent      Rcvd
    Opens:                           1         1
    Notifications:                   0         0
    Updates:                         4         4
    Keepalives:                    235       233
    Enhanced Route Refresh:          0         0
    Begin of Route Refresh:          0         0
    End of Route Refresh:            0         0
    Total messages:                240       238
  Prefix Statistics:
                                   Sent      Rcvd     Best Paths     Best ECMP Paths
    IPv4 Unicast:                     3         1              1                   0
    IPv6 Unicast:                     0         0              0                   0
  Configured maximum total number of routes is 256000, warning limit is 204800
  Inbound updates dropped by reason:
    AS path loop detection: 2
    Cluster ID loop detection: 0
    Enforced First AS: 0
    Malformed MPBGP routes: 0
    Originator ID matches local router ID: 0
    Nexthop matches local IP address: 0
    Unexpected IPv6 nexthop for IPv4 routes: 0
  Inbound updates with attribute errors:
    Resulting in removal of all paths in update (treat as withdraw): 0
    Resulting in AFI/SAFI disable: 0
    Resulting in attribute ignore: 0
    Disabled AFI/SAFIs: None
  Inbound paths dropped by reason:
    IPv4 unicast NLRIs dropped due to martian prefix: 0
    IPv6 unicast NLRIs dropped due to martian prefix: 0
    IPv4 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv4 labeled-unicast NLRIs dropped due to martian prefix: 0
    IPv6 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv6 labeled-unicast NLRIs dropped due to martian prefix: 0
    VPN-IPv4 NLRIs dropped due to route import match failure: 0
    VPN-IPv6 NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to unsupported route type: 0
    Link-state NLRIs dropped because reception is unsupported: 0
    RT Membership NLRIs dropped due to local origin ASN received from external peer: 0
  Outbound paths dropped by reason:
    IPv4 local address not available: 0
    IPv6 local address not available: 0
Local AS is 65100, local router ID 10.1.0.2
TTL is 1
Local TCP address is fe80::5231:cff:fe05:79b7, local port is 34149
Remote TCP address is fe80::52fc:7cff:fed4:3ea7, remote port is 179
Local next hop for next hop self:
  IPv4 Unicast: missing (cannot send updates with next hop self)
BFD is enabled and state is Up
MD5 authentication is enabled
! Sending extended community not configured, updates will be sent without extended communities or route targets
TCP Socket Information:
  TCP state is ESTABLISHED
  Recv-Q: 0/32768
  Send-Q: 0/46080
  Outgoing Maximum Segment Size (MSS): 1440
  Total Number of TCP retransmissions: 0
  Options:
    Timestamps enabled: no
    Selective Acknowledgments enabled: yes
    Window Scale enabled: yes
    Explicit Congestion Notification (ECN) enabled: no
  Socket Statistics:
    Window Scale (wscale): 7,7
    Retransmission Timeout (rto): 204.0ms
    Round-trip Time (rtt/rtvar): 3.9ms/0.8ms
    Delayed Ack Timeout (ato): 40.0ms
    Congestion Window (cwnd): 10
    TCP Throughput: 29.77 Mbps
    Advertised Recv Window (rcv_space): 14400
```

<!-- TOC --><a name="bgp-rib-1"></a>
#### BGP RIB
Видим, что маршруты до нужных loopback'ов доступны. AS-PATH выглядят корректными.

```
Spine-2#show bgp ipv4 unicast
BGP routing table information for VRF default
Router identifier 10.1.0.2, local AS number 65100
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.2/32            -                     -       -          -       0       i
 * >      10.1.1.1/32            fe80::52b7:2fff:fec9:d5a9%Et1 0       -          100     0       65001 i
 * >      10.1.1.2/32            fe80::52e1:2eff:feb6:c5d%Et2 0       -          100     0       65002 i
 * >      10.1.1.3/32            fe80::52fc:7cff:fed4:3ea7%Et3 0       -          100     0       65003 i
 ```

 #### Route Table
Маршруты до Loopback'ов Leaf'ов в таблице маршрутизации присутствуют.

 ```
Spine-2#show ip route

VRF: default
Codes: C - connected, S - static, K - kernel, 
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 C        10.1.0.2/32 [0/0]
           via Loopback0, directly connected
 B E      10.1.1.1/32 [200/0]
           via fe80::52b7:2fff:fec9:d5a9, Ethernet1
 B E      10.1.1.2/32 [200/0]
           via fe80::52e1:2eff:feb6:c5d, Ethernet2
 B E      10.1.1.3/32 [200/0]
           via fe80::52fc:7cff:fed4:3ea7, Ethernet3
```

<!-- TOC --><a name="bfd-peers-1"></a>
#### BFD peers
В последнюю очередь проверим состояние наших BFD-сессий. Вроде бы все выглядит в порядке.

```
Spine-2#show bfd peers
VRF name: default
-----------------
DstAddr                        MyDisc    YourDisc  Interface/Transport    Type 
-------------------------- ----------- ----------- -------------------- -------
fe80::52b7:2fff:fec9:d5a9   367494668   121294589        Ethernet1(13)  normal 
fe80::52e1:2eff:feb6:c5d   3292986894  3513218795        Ethernet2(14)  normal 
fe80::52fc:7cff:fed4:3ea7  2943854172  2226934466        Ethernet3(15)  normal 

           LastUp       LastDown            LastDiag    State
-------------------- -------------- ------------------- -----
   05/29/24 07:24             NA       No Diagnostic       Up
   05/29/24 07:24             NA       No Diagnostic       Up
   05/29/24 07:24             NA       No Diagnostic       Up
```

<!-- TOC --><a name="-leaf-1-1"></a>
### Верификация Leaf-1
<!-- TOC --><a name="-6"></a>
#### Интерфейсы
Проверим IPv4 и IPv6 адресации

```
Leaf-1#show ip interface brief
                                                                        Address
Interface        IP Address       Status      Protocol           MTU    Owner  
---------------- ---------------- ----------- ------------- ----------- -------
Ethernet1        unassigned       up          up                1500           
Ethernet2        unassigned       up          up                1500           
Loopback0        10.1.1.1/32      up          up               65535           
Management1      unassigned       down        down              1500           
```

```
Leaf-1#show ipv6 interface brief
Interface  Status   MTU  IPv6 Address                   Addr State  Addr Source
---------- ------- ----- ----------------------------- ------------ -----------
Et1        up      1500  fe80::52b7:2fff:fec9:d5a9/64   up          link local 
Et2        up      1500  fe80::52b7:2fff:fec9:d5a9/64   up          link local 
```

Все выглядит в порядке. Link-local адреса на месте, IPv4-loopback присутствует

<!-- TOC --><a name="route-map"></a>
#### Route-map
Проверим, что маршрутная карта для редистрибьюции создана корректно.

```     
Leaf-1#show route-map BGP_REDISTRIBUTE_CONNECTED
route-map BGP_REDISTRIBUTE_CONNECTED permit 10
  Description:
  Match clauses:
    match interface Loopback0
  SubRouteMap:
  Set clauses:
```

<!-- TOC --><a name="bgp-instance-2"></a>
#### BGP instance
Здесь можно посмотреть общую информацию, относящуюся к нашему инстансу BGP. Все выглядит корректно. Видим, что на редистрибьюцию connected-маршрутов в IPv4 Unicast применяется наша маршрутная карта.

```
Leaf-1#show bgp instance
BGP instance information for VRF default
BGP Local AS: 65001, Router ID: 10.1.1.1
Total peers:             2
  Static peers:          2
  Dynamic peers:         0
  Disabled peers:        0
  Established peers:     2
Four Octet ASN mode enabled
Graceful restart helper mode enabled
Graceful restart mode disabled
Graceful restart timer timeout: 00:05:00
End of rib timer timeout: 00:05:00
Attributes of the reflected routes are not preserved
Number of Adj-RIB-Ins awaiting cleanup: 0
UCMP mode: disabled
Peer mac resolution timeout: 00:00:00
Next hop labeled unicast origination LFIB uninstall delay timer: 04:00:00
BGP IPv4 Listen Port Status: listening on port 179
BGP IPv6 Listen Port Status: listening on port 179
BGP Convergence information:
    BGP has converged:   yes,   Time taken to converge: 00:00:01
    Outstanding EORs:    0,     Outstanding Keepalives: 0
    Convergence timeout: 00:05:00
BGP Convergence timer is inactive
BGP Convergence slow-peer timeout: 00:01:30
Address family IPv4 Unicast:
  Redistributed routes into BGP:
    Connected, Route map: BGP_REDISTRIBUTE_CONNECTED
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Convergence based update synchronization is disabled
  Extended next-hop capability is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
Address family IPv4 MplsLabel:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
  Target RIBs: Tunnel RIB
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family IPv4 MplsVpn:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family IPv6 Unicast:
  Redistributed routes into BGP:
    Connected, Route map: BGP_REDISTRIBUTE_CONNECTED
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  6PE Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib
  6PE Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
Address family IPv6 MplsLabel:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
  Target RIBs: Tunnel RIB
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family IPv6 MplsVpn:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family L2VPN VPLS:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
Address family L2VPN EVPN:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  VXLAN Resolution RIBs: system-unicast-rib
  VXLAN Default Resolution RIBs: system-unicast-rib
  MPLS Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  MPLS Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
```

<!-- TOC --><a name="bgp-summary-2"></a>
#### BGP summary
Здесь мы видим, что наши соседства с Spine-коммутаторами установлены. Номера AS корректные, локальный RouterID берется из Loopback0, состояние сессий Established, AFI/SAFI верный - IPv4 Unicast.
```
Leaf-1#show bgp summary
BGP summary information for VRF default
Router identifier 10.1.1.1, local AS number 65001
Neighbor                               AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
----------------------------- ----------- ------------- ----------------------- -------------- ---------- ----------
fe80::5231:cff:fe05:79b7%Et2        65100 Established   IPv4 Unicast            Negotiated              3          3
fe80::5284:b8ff:fe48:4dcb%Et1       65100 Established   IPv4 Unicast            Negotiated              3          3
```

<!-- TOC --><a name="bgp-neighbors-detail-2"></a>
#### BGP Neighbors (detail)
Здесь мы можем убедиться, что соседи наследуют конфигурацию из пир-группы SPINES, keepalive и hold таймеры составляют 3 и 9 секунд соответственно. BFD включен для каждого соседа и находится в состоянии Up. Аутентификация включена.

```
Leaf-1#show bgp neighbors
BGP neighbor is fe80::5231:cff:fe05:79b7%Et2, remote AS 65100, external link
 LLDP Neighbors: Spine-2 (Arista Networks EOS version 4.29.2F running on an Arista vEOS-lab)
  BGP version 4, remote router ID 10.1.0.2, VRF default
  Inherits configuration from and member of peer-group UNDERLAY
  Last read 00:00:01, last write 00:00:03
  Hold time is 9, keepalive interval is 3 seconds
  Configured hold time is 9, keepalive interval is 3 seconds
  Effective minimum hold time is 3 seconds
  Hold timer is active, time left: 00:00:08
  Keepalive timer is inactive
  Connect timer is inactive
  Idle-restart timer is inactive
  BGP state is Established, up for 00:11:51
  Number of transitions to established: 1
  Last state was OpenConfirm
  Last event was Established
  Types of communities advertised: none
  Enhanced route refresh stale path removal disabled
  Outbound enhanced route refresh enabled
  Neighbor Capabilities:
    Multiprotocol IPv4 Unicast: advertised and received and negotiated
    Four Octet ASN: advertised and received and negotiated
    Route Refresh: advertised and received and negotiated
    Enhanced route refresh: advertised and received and negotiated
    Send End-of-RIB messages: advertised and received and negotiated
    Additional-paths recv capability:
      IPv4 Unicast: advertised
    Additional-paths send capability:
      IPv4 Unicast: received
    Extended Next-Hop Capability:
      IPv4 Unicast: advertised and received and negotiated
  Restart timer is inactive
  End of rib timer is inactive
    IPv4 Unicast End-of-RIB received: Yes
      Received 00:10:32
      Number of stale paths removed after graceful restart: 0
  AIGP attribute send and receive for IPv4 Unicast are disabled
  AIGP attribute send and receive for IPv4 with MPLS Labels are disabled
  AIGP attribute send and receive for IPv6 Unicast are disabled
  AIGP attribute send and receive for IPv6 with MPLS Labels are disabled
  BGP session driven failover for IPv6 Unicast is disabled
  BGP session driven failover for IPv4 Unicast is disabled
  Message Statistics:
                                  Sent      Rcvd
    Opens:                           1         1
    Notifications:                   0         0
    Updates:                         3         4
    Keepalives:                    280       278
    Enhanced Route Refresh:          0         0
    Begin of Route Refresh:          0         0
    End of Route Refresh:            0         0
    Total messages:                284       283
  Prefix Statistics:
                                   Sent      Rcvd     Best Paths     Best ECMP Paths
    IPv4 Unicast:                     2         3              3                   2
    IPv6 Unicast:                     0         0              0                   0
  Configured maximum total number of routes is 256000, warning limit is 204800
  Inbound updates dropped by reason:
    AS path loop detection: 0
    Cluster ID loop detection: 0
    Enforced First AS: 0
    Malformed MPBGP routes: 0
    Originator ID matches local router ID: 0
    Nexthop matches local IP address: 0
    Unexpected IPv6 nexthop for IPv4 routes: 0
  Inbound updates with attribute errors:
    Resulting in removal of all paths in update (treat as withdraw): 0
    Resulting in AFI/SAFI disable: 0
    Resulting in attribute ignore: 0
    Disabled AFI/SAFIs: None
  Inbound paths dropped by reason:
    IPv4 unicast NLRIs dropped due to martian prefix: 0
    IPv6 unicast NLRIs dropped due to martian prefix: 0
    IPv4 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv4 labeled-unicast NLRIs dropped due to martian prefix: 0
    IPv6 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv6 labeled-unicast NLRIs dropped due to martian prefix: 0
    VPN-IPv4 NLRIs dropped due to route import match failure: 0
    VPN-IPv6 NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to unsupported route type: 0
    Link-state NLRIs dropped because reception is unsupported: 0
    RT Membership NLRIs dropped due to local origin ASN received from external peer: 0
  Outbound paths dropped by reason:
    IPv4 local address not available: 0
    IPv6 local address not available: 0
Local AS is 65001, local router ID 10.1.1.1
TTL is 1
Local TCP address is fe80::52b7:2fff:fec9:d5a9, local port is 179
Remote TCP address is fe80::5231:cff:fe05:79b7, remote port is 40569
Local next hop for next hop self:
  IPv4 Unicast: missing (cannot send updates with next hop self)
BFD is enabled and state is Up
MD5 authentication is enabled
! Sending extended community not configured, updates will be sent without extended communities or route targets
TCP Socket Information:
  TCP state is ESTABLISHED
  Recv-Q: 0/32768
  Send-Q: 0/46080
  Outgoing Maximum Segment Size (MSS): 1420
  Total Number of TCP retransmissions: 0
  Options:
    Timestamps enabled: no
    Selective Acknowledgments enabled: yes
    Window Scale enabled: yes
    Explicit Congestion Notification (ECN) enabled: no
  Socket Statistics:
    Window Scale (wscale): 7,7
    Retransmission Timeout (rto): 204.0ms
    Round-trip Time (rtt/rtvar): 3.7ms/0.6ms
    Delayed Ack Timeout (ato): 40.0ms
    Congestion Window (cwnd): 10
    TCP Throughput: 30.79 Mbps
    Advertised Recv Window (rcv_space): 14400

BGP neighbor is fe80::5284:b8ff:fe48:4dcb%Et1, remote AS 65100, external link
 LLDP Neighbors: Spine-1 (Arista Networks EOS version 4.29.2F running on an Arista vEOS-lab)
  BGP version 4, remote router ID 10.1.0.1, VRF default
  Inherits configuration from and member of peer-group UNDERLAY
  Last read 00:00:02, last write 00:00:02
  Hold time is 9, keepalive interval is 3 seconds
  Configured hold time is 9, keepalive interval is 3 seconds
  Effective minimum hold time is 3 seconds
  Hold timer is active, time left: 00:00:07
  Keepalive timer is inactive
  Connect timer is inactive
  Idle-restart timer is inactive
  BGP state is Established, up for 00:11:50
  Number of transitions to established: 1
  Last state was OpenConfirm
  Last event was Established
  Types of communities advertised: none
  Enhanced route refresh stale path removal disabled
  Outbound enhanced route refresh enabled
  Neighbor Capabilities:
    Multiprotocol IPv4 Unicast: advertised and received and negotiated
    Four Octet ASN: advertised and received and negotiated
    Route Refresh: advertised and received and negotiated
    Enhanced route refresh: advertised and received and negotiated
    Send End-of-RIB messages: advertised and received and negotiated
    Additional-paths recv capability:
      IPv4 Unicast: advertised
    Additional-paths send capability:
      IPv4 Unicast: received
    Extended Next-Hop Capability:
      IPv4 Unicast: advertised and received and negotiated
  Restart timer is inactive
  End of rib timer is inactive
    IPv4 Unicast End-of-RIB received: Yes
      Received 00:10:33
      Number of stale paths removed after graceful restart: 0
  AIGP attribute send and receive for IPv4 Unicast are disabled
  AIGP attribute send and receive for IPv4 with MPLS Labels are disabled
  AIGP attribute send and receive for IPv6 Unicast are disabled
  AIGP attribute send and receive for IPv6 with MPLS Labels are disabled
  BGP session driven failover for IPv6 Unicast is disabled
  BGP session driven failover for IPv4 Unicast is disabled
  Message Statistics:
                                  Sent      Rcvd
    Opens:                           1         1
    Notifications:                   0         0
    Updates:                         5         4
    Keepalives:                    282       278
    Enhanced Route Refresh:          0         0
    Begin of Route Refresh:          0         0
    End of Route Refresh:            0         0
    Total messages:                288       283
  Prefix Statistics:
                                   Sent      Rcvd     Best Paths     Best ECMP Paths
    IPv4 Unicast:                     4         3              3                   0
    IPv6 Unicast:                     0         0              0                   0
  Configured maximum total number of routes is 256000, warning limit is 204800
  Inbound updates dropped by reason:
    AS path loop detection: 0
    Cluster ID loop detection: 0
    Enforced First AS: 0
    Malformed MPBGP routes: 0
    Originator ID matches local router ID: 0
    Nexthop matches local IP address: 0
    Unexpected IPv6 nexthop for IPv4 routes: 0
  Inbound updates with attribute errors:
    Resulting in removal of all paths in update (treat as withdraw): 0
    Resulting in AFI/SAFI disable: 0
    Resulting in attribute ignore: 0
    Disabled AFI/SAFIs: None
  Inbound paths dropped by reason:
    IPv4 unicast NLRIs dropped due to martian prefix: 0
    IPv6 unicast NLRIs dropped due to martian prefix: 0
    IPv4 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv4 labeled-unicast NLRIs dropped due to martian prefix: 0
    IPv6 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv6 labeled-unicast NLRIs dropped due to martian prefix: 0
    VPN-IPv4 NLRIs dropped due to route import match failure: 0
    VPN-IPv6 NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to unsupported route type: 0
    Link-state NLRIs dropped because reception is unsupported: 0
    RT Membership NLRIs dropped due to local origin ASN received from external peer: 0
  Outbound paths dropped by reason:
    IPv4 local address not available: 0
    IPv6 local address not available: 0
Local AS is 65001, local router ID 10.1.1.1
TTL is 1
Local TCP address is fe80::52b7:2fff:fec9:d5a9, local port is 179
Remote TCP address is fe80::5284:b8ff:fe48:4dcb, remote port is 35595
Local next hop for next hop self:
  IPv4 Unicast: missing (cannot send updates with next hop self)
BFD is enabled and state is Up
MD5 authentication is enabled
! Sending extended community not configured, updates will be sent without extended communities or route targets
TCP Socket Information:
  TCP state is ESTABLISHED
  Recv-Q: 0/32768
  Send-Q: 0/46080
  Outgoing Maximum Segment Size (MSS): 1420
  Total Number of TCP retransmissions: 0
  Options:
    Timestamps enabled: no
    Selective Acknowledgments enabled: yes
    Window Scale enabled: yes
    Explicit Congestion Notification (ECN) enabled: no
  Socket Statistics:
    Window Scale (wscale): 7,7
    Retransmission Timeout (rto): 208.0ms
    Round-trip Time (rtt/rtvar): 4.0ms/1.0ms
    Delayed Ack Timeout (ato): 40.0ms
    Congestion Window (cwnd): 10
    TCP Throughput: 28.35 Mbps
    Advertised Recv Window (rcv_space): 14400
```

<!-- TOC --><a name="bgp-rib-2"></a>
#### BGP RIB
Видим, что маршруты до нужных loopback'ов доступны. AS-PATH выглядят корректными. ECMP отрабатывает.

```
Leaf-1#show bgp ipv4 unicast
BGP routing table information for VRF default
Router identifier 10.1.1.1, local AS number 65001
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.1/32            fe80::5284:b8ff:fe48:4dcb%Et1 0       -          100     0       65100 i
 * >      10.1.0.2/32            fe80::5231:cff:fe05:79b7%Et2 0       -          100     0       65100 i
 * >      10.1.1.1/32            -                     -       -          -       0       i
 * >Ec    10.1.1.2/32            fe80::5231:cff:fe05:79b7%Et2 0       -          100     0       65100 65002 i
 *  ec    10.1.1.2/32            fe80::5284:b8ff:fe48:4dcb%Et1 0       -          100     0       65100 65002 i
 * >Ec    10.1.1.3/32            fe80::5231:cff:fe05:79b7%Et2 0       -          100     0       65100 65003 i
 *  ec    10.1.1.3/32            fe80::5284:b8ff:fe48:4dcb%Et1 0       -          100     0       65100 65003 i
 ```

 #### Route Table
Маршруты до Loopback'ов Leaf'ов в таблице маршрутизации присутствуют. ECMP работает там, где должен.

 ```
Leaf-1#show ip route

VRF: default
Codes: C - connected, S - static, K - kernel, 
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 B E      10.1.0.1/32 [200/0]
           via fe80::5284:b8ff:fe48:4dcb, Ethernet1
 B E      10.1.0.2/32 [200/0]
           via fe80::5231:cff:fe05:79b7, Ethernet2
 C        10.1.1.1/32 [0/0]
           via Loopback0, directly connected
 B E      10.1.1.2/32 [200/0]
           via fe80::5284:b8ff:fe48:4dcb, Ethernet1
           via fe80::5231:cff:fe05:79b7, Ethernet2
 B E      10.1.1.3/32 [200/0]
           via fe80::5284:b8ff:fe48:4dcb, Ethernet1
           via fe80::5231:cff:fe05:79b7, Ethernet2
```

<!-- TOC --><a name="bfd-peers-2"></a>
#### BFD peers
В последнюю очередь проверим состояние наших BFD-сессий. Вроде бы все выглядит в порядке.

```
Leaf-1#show bfd peers
VRF name: default
-----------------
DstAddr                        MyDisc    YourDisc  Interface/Transport    Type 
-------------------------- ----------- ----------- -------------------- -------
fe80::5231:cff:fe05:79b7    121294589   367494668        Ethernet2(14)  normal 
fe80::5284:b8ff:fe48:4dcb  1408405490  3310161648        Ethernet1(13)  normal 

           LastUp       LastDown            LastDiag    State
-------------------- -------------- ------------------- -----
   05/29/24 07:24             NA       No Diagnostic       Up
   05/29/24 07:24             NA       No Diagnostic       Up
```

<!-- TOC --><a name="-leaf-2-1"></a>
### Верификация Leaf-2
<!-- TOC --><a name="-7"></a>
#### Интерфейсы
Проверим IPv4 и IPv6 адресации

```
Leaf-2#show ip interface brief
                                                                        Address
Interface        IP Address       Status      Protocol           MTU    Owner  
---------------- ---------------- ----------- ------------- ----------- -------
Ethernet1        unassigned       up          up                1500           
Ethernet2        unassigned       up          up                1500           
Loopback0        10.1.1.2/32      up          up               65535           
Management1      unassigned       down        down              1500           
```

```
Leaf-2#show ipv6 interface brief
Interface  Status   MTU   IPv6 Address                  Addr State  Addr Source
---------- ------- ----- ----------------------------- ------------ -----------
Et1        up      1500   fe80::52e1:2eff:feb6:c5d/64   up          link local 
Et2        up      1500   fe80::52e1:2eff:feb6:c5d/64   up          link local 
```

Все выглядит в порядке. Link-local адреса на месте, IPv4-loopback присутствует

<!-- TOC --><a name="route-map-1"></a>
#### Route-map
Проверим, что маршрутная карта для редистрибьюции создана корректно.

```     
Leaf-2#show route-map BGP_REDISTRIBUTE_CONNECTED
route-map BGP_REDISTRIBUTE_CONNECTED permit 10
  Description:
  Match clauses:
    match interface Loopback0
  SubRouteMap:
  Set clauses:
```

<!-- TOC --><a name="bgp-instance-3"></a>
#### BGP instance
Здесь можно посмотреть общую информацию, относящуюся к нашему инстансу BGP. Все выглядит корректно. Видим, что на редистрибьюцию connected-маршрутов в IPv4 Unicast применяется наша маршрутная карта.

```
Leaf-2#show bgp instance 
BGP instance information for VRF default
BGP Local AS: 65002, Router ID: 10.1.1.2
Total peers:             2
  Static peers:          2
  Dynamic peers:         0
  Disabled peers:        0
  Established peers:     2
Four Octet ASN mode enabled
Graceful restart helper mode enabled
Graceful restart mode disabled
Graceful restart timer timeout: 00:05:00
End of rib timer timeout: 00:05:00
Attributes of the reflected routes are not preserved
Number of Adj-RIB-Ins awaiting cleanup: 0
UCMP mode: disabled
Peer mac resolution timeout: 00:00:00
Next hop labeled unicast origination LFIB uninstall delay timer: 04:00:00
BGP IPv4 Listen Port Status: listening on port 179
BGP IPv6 Listen Port Status: listening on port 179
BGP Convergence information:
    BGP has converged:   yes,   Time taken to converge: 00:00:02
    Outstanding EORs:    0,     Outstanding Keepalives: 0
    Convergence timeout: 00:05:00
BGP Convergence timer is inactive
BGP Convergence slow-peer timeout: 00:01:30
Address family IPv4 Unicast:
  Redistributed routes into BGP:
    Connected, Route map: BGP_REDISTRIBUTE_CONNECTED
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Convergence based update synchronization is disabled
  Extended next-hop capability is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
Address family IPv4 MplsLabel:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
  Target RIBs: Tunnel RIB
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family IPv4 MplsVpn:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family IPv6 Unicast:
  Redistributed routes into BGP:
    Connected, Route map: BGP_REDISTRIBUTE_CONNECTED
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  6PE Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib
  6PE Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
Address family IPv6 MplsLabel:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
  Target RIBs: Tunnel RIB
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family IPv6 MplsVpn:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family L2VPN VPLS:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
Address family L2VPN EVPN:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  VXLAN Resolution RIBs: system-unicast-rib
  VXLAN Default Resolution RIBs: system-unicast-rib
  MPLS Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  MPLS Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
```

<!-- TOC --><a name="bgp-summary-3"></a>
#### BGP summary
Здесь мы видим, что наши соседства с Spine-коммутаторами установлены. Номера AS корректные, локальный RouterID берется из Loopback0, состояние сессий Established, AFI/SAFI верный - IPv4 Unicast.
```
Leaf-2#show bgp summary
BGP summary information for VRF default
Router identifier 10.1.1.2, local AS number 65002
Neighbor                               AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
----------------------------- ----------- ------------- ----------------------- -------------- ---------- ----------
fe80::5231:cff:fe05:79b7%Et2        65100 Established   IPv4 Unicast            Negotiated              3          3
fe80::5284:b8ff:fe48:4dcb%Et1       65100 Established   IPv4 Unicast            Negotiated              3          3
```

<!-- TOC --><a name="bgp-neighbors-detail-3"></a>
#### BGP Neighbors (detail)
Здесь мы можем убедиться, что соседи наследуют конфигурацию из пир-группы SPINES, keepalive и hold таймеры составляют 3 и 9 секунд соответственно. BFD включен для каждого соседа и находится в состоянии Up. Аутентификация включена.

```
Leaf-2#show bgp neighbors
BGP neighbor is fe80::5231:cff:fe05:79b7%Et2, remote AS 65100, external link
 LLDP Neighbors: Spine-2 (Arista Networks EOS version 4.29.2F running on an Arista vEOS-lab)
  BGP version 4, remote router ID 10.1.0.2, VRF default
  Inherits configuration from and member of peer-group UNDERLAY
  Last read 00:00:01, last write 00:00:01
  Hold time is 9, keepalive interval is 3 seconds
  Configured hold time is 9, keepalive interval is 3 seconds
  Effective minimum hold time is 3 seconds
  Hold timer is active, time left: 00:00:08
  Keepalive timer is active, time left: 00:00:01
  Connect timer is inactive
  Idle-restart timer is inactive
  BGP state is Established, up for 00:13:56
  Number of transitions to established: 1
  Last state was OpenConfirm
  Last event was Established
  Types of communities advertised: none
  Enhanced route refresh stale path removal disabled
  Outbound enhanced route refresh enabled
  Neighbor Capabilities:
    Multiprotocol IPv4 Unicast: advertised and received and negotiated
    Four Octet ASN: advertised and received and negotiated
    Route Refresh: advertised and received and negotiated
    Enhanced route refresh: advertised and received and negotiated
    Send End-of-RIB messages: advertised and received and negotiated
    Additional-paths recv capability:
      IPv4 Unicast: advertised
    Additional-paths send capability:
      IPv4 Unicast: received
    Extended Next-Hop Capability:
      IPv4 Unicast: advertised and received and negotiated
  Restart timer is inactive
  End of rib timer is inactive
    IPv4 Unicast End-of-RIB received: Yes
      Received 00:12:26
      Number of stale paths removed after graceful restart: 0
  AIGP attribute send and receive for IPv4 Unicast are disabled
  AIGP attribute send and receive for IPv4 with MPLS Labels are disabled
  AIGP attribute send and receive for IPv6 Unicast are disabled
  AIGP attribute send and receive for IPv6 with MPLS Labels are disabled
  BGP session driven failover for IPv6 Unicast is disabled
  BGP session driven failover for IPv4 Unicast is disabled
  Message Statistics:
                                  Sent      Rcvd
    Opens:                           1         1
    Notifications:                   0         0
    Updates:                         4         4
    Keepalives:                    332       330
    Enhanced Route Refresh:          0         0
    Begin of Route Refresh:          0         0
    End of Route Refresh:            0         0
    Total messages:                337       335
  Prefix Statistics:
                                   Sent      Rcvd     Best Paths     Best ECMP Paths
    IPv4 Unicast:                     3         3              3                   1
    IPv6 Unicast:                     0         0              0                   0
  Configured maximum total number of routes is 256000, warning limit is 204800
  Inbound updates dropped by reason:
    AS path loop detection: 0
    Cluster ID loop detection: 0
    Enforced First AS: 0
    Malformed MPBGP routes: 0
    Originator ID matches local router ID: 0
    Nexthop matches local IP address: 0
    Unexpected IPv6 nexthop for IPv4 routes: 0
  Inbound updates with attribute errors:
    Resulting in removal of all paths in update (treat as withdraw): 0
    Resulting in AFI/SAFI disable: 0
    Resulting in attribute ignore: 0
    Disabled AFI/SAFIs: None
  Inbound paths dropped by reason:
    IPv4 unicast NLRIs dropped due to martian prefix: 0
    IPv6 unicast NLRIs dropped due to martian prefix: 0
    IPv4 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv4 labeled-unicast NLRIs dropped due to martian prefix: 0
    IPv6 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv6 labeled-unicast NLRIs dropped due to martian prefix: 0
    VPN-IPv4 NLRIs dropped due to route import match failure: 0
    VPN-IPv6 NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to unsupported route type: 0
    Link-state NLRIs dropped because reception is unsupported: 0
    RT Membership NLRIs dropped due to local origin ASN received from external peer: 0
  Outbound paths dropped by reason:
    IPv4 local address not available: 0
    IPv6 local address not available: 0
Local AS is 65002, local router ID 10.1.1.2
TTL is 1
Local TCP address is fe80::52e1:2eff:feb6:c5d, local port is 40671
Remote TCP address is fe80::5231:cff:fe05:79b7, remote port is 179
Local next hop for next hop self:
  IPv4 Unicast: missing (cannot send updates with next hop self)
BFD is enabled and state is Up
MD5 authentication is enabled
! Sending extended community not configured, updates will be sent without extended communities or route targets
TCP Socket Information:
  TCP state is ESTABLISHED
  Recv-Q: 0/32768
  Send-Q: 0/46080
  Outgoing Maximum Segment Size (MSS): 1440
  Total Number of TCP retransmissions: 2
  Options:
    Timestamps enabled: no
    Selective Acknowledgments enabled: yes
    Window Scale enabled: yes
    Explicit Congestion Notification (ECN) enabled: no
  Socket Statistics:
    Window Scale (wscale): 7,7
    Retransmission Timeout (rto): 212.0ms
    Round-trip Time (rtt/rtvar): 9.1ms/10.8ms
    Delayed Ack Timeout (ato): 40.0ms
    Congestion Window (cwnd): 2
    TCP Throughput: 2.53 Mbps
    Advertised Recv Window (rcv_space): 14400

BGP neighbor is fe80::5284:b8ff:fe48:4dcb%Et1, remote AS 65100, external link
 LLDP Neighbors: Spine-1 (Arista Networks EOS version 4.29.2F running on an Arista vEOS-lab)
  BGP version 4, remote router ID 10.1.0.1, VRF default
  Inherits configuration from and member of peer-group UNDERLAY
  Last read 00:00:01, last write 00:00:01
  Hold time is 9, keepalive interval is 3 seconds
  Configured hold time is 9, keepalive interval is 3 seconds
  Effective minimum hold time is 3 seconds
  Hold timer is active, time left: 00:00:08
  Keepalive timer is active, time left: 00:00:01
  Connect timer is inactive
  Idle-restart timer is inactive
  BGP state is Established, up for 00:13:57
  Number of transitions to established: 1
  Last state was OpenConfirm
  Last event was Established
  Types of communities advertised: none
  Enhanced route refresh stale path removal disabled
  Outbound enhanced route refresh enabled
  Neighbor Capabilities:
    Multiprotocol IPv4 Unicast: advertised and received and negotiated
    Four Octet ASN: advertised and received and negotiated
    Route Refresh: advertised and received and negotiated
    Enhanced route refresh: advertised and received and negotiated
    Send End-of-RIB messages: advertised and received and negotiated
    Additional-paths recv capability:
      IPv4 Unicast: advertised
    Additional-paths send capability:
      IPv4 Unicast: received
    Extended Next-Hop Capability:
      IPv4 Unicast: advertised and received and negotiated
  Restart timer is inactive
  End of rib timer is inactive
    IPv4 Unicast End-of-RIB received: Yes
      Received 00:12:27
      Number of stale paths removed after graceful restart: 0
  AIGP attribute send and receive for IPv4 Unicast are disabled
  AIGP attribute send and receive for IPv4 with MPLS Labels are disabled
  AIGP attribute send and receive for IPv6 Unicast are disabled
  AIGP attribute send and receive for IPv6 with MPLS Labels are disabled
  BGP session driven failover for IPv6 Unicast is disabled
  BGP session driven failover for IPv4 Unicast is disabled
  Message Statistics:
                                  Sent      Rcvd
    Opens:                           1         1
    Notifications:                   0         0
    Updates:                         4         4
    Keepalives:                    330       328
    Enhanced Route Refresh:          0         0
    Begin of Route Refresh:          0         0
    End of Route Refresh:            0         0
    Total messages:                335       333
  Prefix Statistics:
                                   Sent      Rcvd     Best Paths     Best ECMP Paths
    IPv4 Unicast:                     3         3              3                   1
    IPv6 Unicast:                     0         0              0                   0
  Configured maximum total number of routes is 256000, warning limit is 204800
  Inbound updates dropped by reason:
    AS path loop detection: 0
    Cluster ID loop detection: 0
    Enforced First AS: 0
    Malformed MPBGP routes: 0
    Originator ID matches local router ID: 0
    Nexthop matches local IP address: 0
    Unexpected IPv6 nexthop for IPv4 routes: 0
  Inbound updates with attribute errors:
    Resulting in removal of all paths in update (treat as withdraw): 0
    Resulting in AFI/SAFI disable: 0
    Resulting in attribute ignore: 0
    Disabled AFI/SAFIs: None
  Inbound paths dropped by reason:
    IPv4 unicast NLRIs dropped due to martian prefix: 0
    IPv6 unicast NLRIs dropped due to martian prefix: 0
    IPv4 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv4 labeled-unicast NLRIs dropped due to martian prefix: 0
    IPv6 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv6 labeled-unicast NLRIs dropped due to martian prefix: 0
    VPN-IPv4 NLRIs dropped due to route import match failure: 0
    VPN-IPv6 NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to unsupported route type: 0
    Link-state NLRIs dropped because reception is unsupported: 0
    RT Membership NLRIs dropped due to local origin ASN received from external peer: 0
  Outbound paths dropped by reason:
    IPv4 local address not available: 0
    IPv6 local address not available: 0
Local AS is 65002, local router ID 10.1.1.2
TTL is 1
Local TCP address is fe80::52e1:2eff:feb6:c5d, local port is 37385
Remote TCP address is fe80::5284:b8ff:fe48:4dcb, remote port is 179
Local next hop for next hop self:
  IPv4 Unicast: missing (cannot send updates with next hop self)
BFD is enabled and state is Up
MD5 authentication is enabled
! Sending extended community not configured, updates will be sent without extended communities or route targets
TCP Socket Information:
  TCP state is ESTABLISHED
  Recv-Q: 0/32768
  Send-Q: 0/46080
  Outgoing Maximum Segment Size (MSS): 1440
  Total Number of TCP retransmissions: 0
  Options:
    Timestamps enabled: no
    Selective Acknowledgments enabled: yes
    Window Scale enabled: yes
    Explicit Congestion Notification (ECN) enabled: no
  Socket Statistics:
    Window Scale (wscale): 7,7
    Retransmission Timeout (rto): 204.0ms
    Round-trip Time (rtt/rtvar): 3.9ms/0.7ms
    Delayed Ack Timeout (ato): 40.0ms
    Congestion Window (cwnd): 10
    TCP Throughput: 29.69 Mbps
    Advertised Recv Window (rcv_space): 14400
```

<!-- TOC --><a name="bgp-rib-3"></a>
#### BGP RIB
Видим, что маршруты до нужных loopback'ов доступны. AS-PATH выглядят корректными. ECMP отрабатывает.

```
Leaf-2#show bgp ipv4 unicast
BGP routing table information for VRF default
Router identifier 10.1.1.2, local AS number 65002
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.1/32            fe80::5284:b8ff:fe48:4dcb%Et1 0       -          100     0       65100 i
 * >      10.1.0.2/32            fe80::5231:cff:fe05:79b7%Et2 0       -          100     0       65100 i
 * >Ec    10.1.1.1/32            fe80::5231:cff:fe05:79b7%Et2 0       -          100     0       65100 65001 i
 *  ec    10.1.1.1/32            fe80::5284:b8ff:fe48:4dcb%Et1 0       -          100     0       65100 65001 i
 * >      10.1.1.2/32            -                     -       -          -       0       i
 * >Ec    10.1.1.3/32            fe80::5284:b8ff:fe48:4dcb%Et1 0       -          100     0       65100 65003 i
 *  ec    10.1.1.3/32            fe80::5231:cff:fe05:79b7%Et2 0       -          100     0       65100 65003 i
 ```

 #### Route Table
Маршруты до Loopback'ов Leaf'ов в таблице маршрутизации присутствуют. ECMP работает там, где должен.

 ```
Leaf-2#show ip route

VRF: default
Codes: C - connected, S - static, K - kernel, 
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 B E      10.1.0.1/32 [200/0]
           via fe80::5284:b8ff:fe48:4dcb, Ethernet1
 B E      10.1.0.2/32 [200/0]
           via fe80::5231:cff:fe05:79b7, Ethernet2
 B E      10.1.1.1/32 [200/0]
           via fe80::5284:b8ff:fe48:4dcb, Ethernet1
           via fe80::5231:cff:fe05:79b7, Ethernet2
 C        10.1.1.2/32 [0/0]
           via Loopback0, directly connected
 B E      10.1.1.3/32 [200/0]
           via fe80::5284:b8ff:fe48:4dcb, Ethernet1
           via fe80::5231:cff:fe05:79b7, Ethernet2
```

<!-- TOC --><a name="bfd-peers-3"></a>
#### BFD peers
В последнюю очередь проверим состояние наших BFD-сессий. Вроде бы все выглядит в порядке.

```
Leaf-2#show bfd peers
VRF name: default
-----------------
DstAddr                        MyDisc    YourDisc  Interface/Transport    Type 
-------------------------- ----------- ----------- -------------------- -------
fe80::5231:cff:fe05:79b7   3513218795  3292986894        Ethernet2(14)  normal 
fe80::5284:b8ff:fe48:4dcb   350688201  3140695752        Ethernet1(13)  normal 

           LastUp       LastDown            LastDiag    State
-------------------- -------------- ------------------- -----
   05/29/24 07:24             NA       No Diagnostic       Up
   05/29/24 07:24             NA       No Diagnostic       Up
```

<!-- TOC --><a name="-leaf-3-1"></a>
### Верификация Leaf-3
<!-- TOC --><a name="-8"></a>
#### Интерфейсы
Проверим IPv4 и IPv6 адресации

```
Leaf-3#show ip interface brief
                                                                        Address
Interface        IP Address       Status      Protocol           MTU    Owner  
---------------- ---------------- ----------- ------------- ----------- -------
Ethernet1        unassigned       up          up                1500           
Ethernet2        unassigned       up          up                1500           
Loopback0        10.1.1.3/32      up          up               65535           
Management1      unassigned       down        down              1500           
```

```
Leaf-3#show ipv6 interface brief
Interface  Status   MTU  IPv6 Address                   Addr State  Addr Source
---------- ------- ----- ----------------------------- ------------ -----------
Et1        up      1500  fe80::52fc:7cff:fed4:3ea7/64   up          link local 
Et2        up      1500  fe80::52fc:7cff:fed4:3ea7/64   up          link local 
```

Все выглядит в порядке. Link-local адреса на месте, IPv4-loopback присутствует

<!-- TOC --><a name="route-map-2"></a>
#### Route-map
Проверим, что маршрутная карта для редистрибьюции создана корректно.

```     
Leaf-3#show route-map BGP_REDISTRIBUTE_CONNECTED
route-map BGP_REDISTRIBUTE_CONNECTED permit 10
  Description:
  Match clauses:
    match interface Loopback0
  SubRouteMap:
  Set clauses:
```

<!-- TOC --><a name="bgp-instance-4"></a>
#### BGP instance
Здесь можно посмотреть общую информацию, относящуюся к нашему инстансу BGP. Все выглядит корректно. Видим, что на редистрибьюцию connected-маршрутов в IPv4 Unicast применяется наша маршрутная карта.

```
Leaf-3#show bgp instance 
BGP instance information for VRF default
BGP Local AS: 65003, Router ID: 10.1.1.3
Total peers:             2
  Static peers:          2
  Dynamic peers:         0
  Disabled peers:        0
  Established peers:     2
Four Octet ASN mode enabled
Graceful restart helper mode enabled
Graceful restart mode disabled
Graceful restart timer timeout: 00:05:00
End of rib timer timeout: 00:05:00
Attributes of the reflected routes are not preserved
Number of Adj-RIB-Ins awaiting cleanup: 0
UCMP mode: disabled
Peer mac resolution timeout: 00:00:00
Next hop labeled unicast origination LFIB uninstall delay timer: 04:00:00
BGP IPv4 Listen Port Status: listening on port 179
BGP IPv6 Listen Port Status: listening on port 179
BGP Convergence information:
    BGP has converged:   yes,   Time taken to converge: 00:00:01
    Outstanding EORs:    0,     Outstanding Keepalives: 0
    Convergence timeout: 00:05:00
BGP Convergence timer is inactive
BGP Convergence slow-peer timeout: 00:01:30
Address family IPv4 Unicast:
  Redistributed routes into BGP:
    Connected, Route map: BGP_REDISTRIBUTE_CONNECTED
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Convergence based update synchronization is disabled
  Extended next-hop capability is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
Address family IPv4 MplsLabel:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
  Target RIBs: Tunnel RIB
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family IPv4 MplsVpn:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family IPv6 Unicast:
  Redistributed routes into BGP:
    Connected, Route map: BGP_REDISTRIBUTE_CONNECTED
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-unicast-rib
  6PE Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib
  6PE Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
Address family IPv6 MplsLabel:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  AIGP session for iBGP peers is enabled
  AIGP session for confed peers is enabled
  AIGP session for eBGP peers is disabled
  Target RIBs: Tunnel RIB
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family IPv6 MplsVpn:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
Address family L2VPN VPLS:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
Address family L2VPN EVPN:
  Additional-paths installation is disabled
  Additional-paths installation with ECMP primary is disabled
  VXLAN Resolution RIBs: system-unicast-rib
  VXLAN Default Resolution RIBs: system-unicast-rib
  MPLS Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
  MPLS Default Resolution RIBs: tunnel-rib colored system-colored-tunnel-rib, tunnel-rib system-tunnel-rib, system-connected
```

<!-- TOC --><a name="bgp-summary-4"></a>
#### BGP summary
Здесь мы видим, что наши соседства с Spine-коммутаторами установлены. Номера AS корректные, локальный RouterID берется из Loopback0, состояние сессий Established, AFI/SAFI верный - IPv4 Unicast.
```
Leaf-3#show bgp summary
BGP summary information for VRF default
Router identifier 10.1.1.3, local AS number 65003
Neighbor                               AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
----------------------------- ----------- ------------- ----------------------- -------------- ---------- ----------
fe80::5231:cff:fe05:79b7%Et2        65100 Established   IPv4 Unicast            Negotiated              3          3
fe80::5284:b8ff:fe48:4dcb%Et1       65100 Established   IPv4 Unicast            Negotiated              3          3
```

<!-- TOC --><a name="bgp-neighbors-detail-4"></a>
#### BGP Neighbors (detail)
Здесь мы можем убедиться, что соседи наследуют конфигурацию из пир-группы SPINES, keepalive и hold таймеры составляют 3 и 9 секунд соответственно. BFD включен для каждого соседа и находится в состоянии Up. Аутентификация включена.

```
Leaf-3#show bgp neighbors
BGP neighbor is fe80::5231:cff:fe05:79b7%Et2, remote AS 65100, external link
 LLDP Neighbors: Spine-2 (Arista Networks EOS version 4.29.2F running on an Arista vEOS-lab)
  BGP version 4, remote router ID 10.1.0.2, VRF default
  Inherits configuration from and member of peer-group UNDERLAY
  Last read 00:00:02, last write 00:00:02
  Hold time is 9, keepalive interval is 3 seconds
  Configured hold time is 9, keepalive interval is 3 seconds
  Effective minimum hold time is 3 seconds
  Hold timer is active, time left: 00:00:07
  Keepalive timer is inactive
  Connect timer is inactive
  Idle-restart timer is inactive
  BGP state is Established, up for 00:15:40
  Number of transitions to established: 1
  Last state was OpenConfirm
  Last event was Established
  Types of communities advertised: none
  Enhanced route refresh stale path removal disabled
  Outbound enhanced route refresh enabled
  Neighbor Capabilities:
    Multiprotocol IPv4 Unicast: advertised and received and negotiated
    Four Octet ASN: advertised and received and negotiated
    Route Refresh: advertised and received and negotiated
    Enhanced route refresh: advertised and received and negotiated
    Send End-of-RIB messages: advertised and received and negotiated
    Additional-paths recv capability:
      IPv4 Unicast: advertised
    Additional-paths send capability:
      IPv4 Unicast: received
    Extended Next-Hop Capability:
      IPv4 Unicast: advertised and received and negotiated
  Restart timer is inactive
  End of rib timer is inactive
    IPv4 Unicast End-of-RIB received: Yes
      Received 00:14:17
      Number of stale paths removed after graceful restart: 0
  AIGP attribute send and receive for IPv4 Unicast are disabled
  AIGP attribute send and receive for IPv4 with MPLS Labels are disabled
  AIGP attribute send and receive for IPv6 Unicast are disabled
  AIGP attribute send and receive for IPv6 with MPLS Labels are disabled
  BGP session driven failover for IPv4 Unicast is disabled
  BGP session driven failover for IPv6 Unicast is disabled
  Message Statistics:
                                  Sent      Rcvd
    Opens:                           1         1
    Notifications:                   0         0
    Updates:                         4         4
    Keepalives:                    370       370
    Enhanced Route Refresh:          0         0
    Begin of Route Refresh:          0         0
    End of Route Refresh:            0         0
    Total messages:                375       375
  Prefix Statistics:
                                   Sent      Rcvd     Best Paths     Best ECMP Paths
    IPv4 Unicast:                     3         3              3                   1
    IPv6 Unicast:                     0         0              0                   0
  Configured maximum total number of routes is 256000, warning limit is 204800
  Inbound updates dropped by reason:
    AS path loop detection: 0
    Cluster ID loop detection: 0
    Enforced First AS: 0
    Malformed MPBGP routes: 0
    Originator ID matches local router ID: 0
    Nexthop matches local IP address: 0
    Unexpected IPv6 nexthop for IPv4 routes: 0
  Inbound updates with attribute errors:
    Resulting in removal of all paths in update (treat as withdraw): 0
    Resulting in AFI/SAFI disable: 0
    Resulting in attribute ignore: 0
    Disabled AFI/SAFIs: None
  Inbound paths dropped by reason:
    IPv4 unicast NLRIs dropped due to martian prefix: 0
    IPv6 unicast NLRIs dropped due to martian prefix: 0
    IPv4 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv4 labeled-unicast NLRIs dropped due to martian prefix: 0
    IPv6 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv6 labeled-unicast NLRIs dropped due to martian prefix: 0
    VPN-IPv4 NLRIs dropped due to route import match failure: 0
    VPN-IPv6 NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to unsupported route type: 0
    Link-state NLRIs dropped because reception is unsupported: 0
    RT Membership NLRIs dropped due to local origin ASN received from external peer: 0
  Outbound paths dropped by reason:
    IPv4 local address not available: 0
    IPv6 local address not available: 0
Local AS is 65003, local router ID 10.1.1.3
TTL is 1
Local TCP address is fe80::52fc:7cff:fed4:3ea7, local port is 179
Remote TCP address is fe80::5231:cff:fe05:79b7, remote port is 34149
Local next hop for next hop self:
  IPv4 Unicast: missing (cannot send updates with next hop self)
BFD is enabled and state is Up
MD5 authentication is enabled
! Sending extended community not configured, updates will be sent without extended communities or route targets
TCP Socket Information:
  TCP state is ESTABLISHED
  Recv-Q: 0/32768
  Send-Q: 0/46080
  Outgoing Maximum Segment Size (MSS): 1420
  Total Number of TCP retransmissions: 0
  Options:
    Timestamps enabled: no
    Selective Acknowledgments enabled: yes
    Window Scale enabled: yes
    Explicit Congestion Notification (ECN) enabled: no
  Socket Statistics:
    Window Scale (wscale): 7,7
    Retransmission Timeout (rto): 204.0ms
    Round-trip Time (rtt/rtvar): 3.3ms/0.2ms
    Delayed Ack Timeout (ato): 40.0ms
    Congestion Window (cwnd): 10
    TCP Throughput: 34.38 Mbps
    Advertised Recv Window (rcv_space): 14400

BGP neighbor is fe80::5284:b8ff:fe48:4dcb%Et1, remote AS 65100, external link
 LLDP Neighbors: Spine-1 (Arista Networks EOS version 4.29.2F running on an Arista vEOS-lab)
  BGP version 4, remote router ID 10.1.0.1, VRF default
  Inherits configuration from and member of peer-group UNDERLAY
  Last read 00:00:02, last write 00:00:01
  Hold time is 9, keepalive interval is 3 seconds
  Configured hold time is 9, keepalive interval is 3 seconds
  Effective minimum hold time is 3 seconds
  Hold timer is active, time left: 00:00:07
  Keepalive timer is active, time left: 00:00:01
  Connect timer is inactive
  Idle-restart timer is inactive
  BGP state is Established, up for 00:15:40
  Number of transitions to established: 1
  Last state was OpenConfirm
  Last event was Established
  Types of communities advertised: none
  Enhanced route refresh stale path removal disabled
  Outbound enhanced route refresh enabled
  Neighbor Capabilities:
    Multiprotocol IPv4 Unicast: advertised and received and negotiated
    Four Octet ASN: advertised and received and negotiated
    Route Refresh: advertised and received and negotiated
    Enhanced route refresh: advertised and received and negotiated
    Send End-of-RIB messages: advertised and received and negotiated
    Additional-paths recv capability:
      IPv4 Unicast: advertised
    Additional-paths send capability:
      IPv4 Unicast: received
    Extended Next-Hop Capability:
      IPv4 Unicast: advertised and received and negotiated
  Restart timer is inactive
  End of rib timer is inactive
    IPv4 Unicast End-of-RIB received: Yes
      Received 00:14:18
      Number of stale paths removed after graceful restart: 0
  AIGP attribute send and receive for IPv4 Unicast are disabled
  AIGP attribute send and receive for IPv4 with MPLS Labels are disabled
  AIGP attribute send and receive for IPv6 Unicast are disabled
  AIGP attribute send and receive for IPv6 with MPLS Labels are disabled
  BGP session driven failover for IPv4 Unicast is disabled
  BGP session driven failover for IPv6 Unicast is disabled
  Message Statistics:
                                  Sent      Rcvd
    Opens:                           1         1
    Notifications:                   0         0
    Updates:                         4         4
    Keepalives:                    369       372
    Enhanced Route Refresh:          0         0
    Begin of Route Refresh:          0         0
    End of Route Refresh:            0         0
    Total messages:                374       377
  Prefix Statistics:
                                   Sent      Rcvd     Best Paths     Best ECMP Paths
    IPv4 Unicast:                     3         3              3                   1
    IPv6 Unicast:                     0         0              0                   0
  Configured maximum total number of routes is 256000, warning limit is 204800
  Inbound updates dropped by reason:
    AS path loop detection: 0
    Cluster ID loop detection: 0
    Enforced First AS: 0
    Malformed MPBGP routes: 0
    Originator ID matches local router ID: 0
    Nexthop matches local IP address: 0
    Unexpected IPv6 nexthop for IPv4 routes: 0
  Inbound updates with attribute errors:
    Resulting in removal of all paths in update (treat as withdraw): 0
    Resulting in AFI/SAFI disable: 0
    Resulting in attribute ignore: 0
    Disabled AFI/SAFIs: None
  Inbound paths dropped by reason:
    IPv4 unicast NLRIs dropped due to martian prefix: 0
    IPv6 unicast NLRIs dropped due to martian prefix: 0
    IPv4 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv4 labeled-unicast NLRIs dropped due to martian prefix: 0
    IPv6 labeled-unicast NLRIs dropped due to excessive labels: 0
    IPv6 labeled-unicast NLRIs dropped due to martian prefix: 0
    VPN-IPv4 NLRIs dropped due to route import match failure: 0
    VPN-IPv6 NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to route import match failure: 0
    L2VPN EVPN NLRIs dropped due to unsupported route type: 0
    Link-state NLRIs dropped because reception is unsupported: 0
    RT Membership NLRIs dropped due to local origin ASN received from external peer: 0
  Outbound paths dropped by reason:
    IPv4 local address not available: 0
    IPv6 local address not available: 0
Local AS is 65003, local router ID 10.1.1.3
TTL is 1
Local TCP address is fe80::52fc:7cff:fed4:3ea7, local port is 44115
Remote TCP address is fe80::5284:b8ff:fe48:4dcb, remote port is 179
Local next hop for next hop self:
  IPv4 Unicast: missing (cannot send updates with next hop self)
BFD is enabled and state is Up
MD5 authentication is enabled
! Sending extended community not configured, updates will be sent without extended communities or route targets
TCP Socket Information:
  TCP state is ESTABLISHED
  Recv-Q: 0/32768
  Send-Q: 0/46080
  Outgoing Maximum Segment Size (MSS): 1440
  Total Number of TCP retransmissions: 3
  Options:
    Timestamps enabled: no
    Selective Acknowledgments enabled: yes
    Window Scale enabled: yes
    Explicit Congestion Notification (ECN) enabled: no
  Socket Statistics:
    Window Scale (wscale): 7,7
    Retransmission Timeout (rto): 204.0ms
    Round-trip Time (rtt/rtvar): 3.9ms/0.9ms
    Delayed Ack Timeout (ato): 40.0ms
    Congestion Window (cwnd): 2
    TCP Throughput: 5.91 Mbps
    Advertised Recv Window (rcv_space): 14400
```

<!-- TOC --><a name="bgp-rib-4"></a>
#### BGP RIB
Видим, что маршруты до нужных loopback'ов доступны. AS-PATH выглядят корректными. ECMP отрабатывает.

```
Leaf-3#show bgp ipv4 unicast
BGP routing table information for VRF default
Router identifier 10.1.1.3, local AS number 65003
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.1/32            fe80::5284:b8ff:fe48:4dcb%Et1 0       -          100     0       65100 i
 * >      10.1.0.2/32            fe80::5231:cff:fe05:79b7%Et2 0       -          100     0       65100 i
 * >Ec    10.1.1.1/32            fe80::5231:cff:fe05:79b7%Et2 0       -          100     0       65100 65001 i
 *  ec    10.1.1.1/32            fe80::5284:b8ff:fe48:4dcb%Et1 0       -          100     0       65100 65001 i
 * >Ec    10.1.1.2/32            fe80::5284:b8ff:fe48:4dcb%Et1 0       -          100     0       65100 65002 i
 *  ec    10.1.1.2/32            fe80::5231:cff:fe05:79b7%Et2 0       -          100     0       65100 65002 i
 * >      10.1.1.3/32            -                     -       -          -       0       i
 ```

 #### Route Table
Маршруты до Loopback'ов Leaf'ов в таблице маршрутизации присутствуют. ECMP работает там, где должен.

 ```
Leaf-3#show ip route

VRF: default
Codes: C - connected, S - static, K - kernel, 
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 B E      10.1.0.1/32 [200/0]
           via fe80::5284:b8ff:fe48:4dcb, Ethernet1
 B E      10.1.0.2/32 [200/0]
           via fe80::5231:cff:fe05:79b7, Ethernet2
 B E      10.1.1.1/32 [200/0]
           via fe80::5284:b8ff:fe48:4dcb, Ethernet1
           via fe80::5231:cff:fe05:79b7, Ethernet2
 B E      10.1.1.2/32 [200/0]
           via fe80::5284:b8ff:fe48:4dcb, Ethernet1
           via fe80::5231:cff:fe05:79b7, Ethernet2
 C        10.1.1.3/32 [0/0]
           via Loopback0, directly connected
```

<!-- TOC --><a name="bfd-peers-4"></a>
#### BFD peers
В последнюю очередь проверим состояние наших BFD-сессий. Вроде бы все выглядит в порядке.

```
Leaf-3#show bfd peers
VRF name: default
-----------------
DstAddr                        MyDisc    YourDisc  Interface/Transport    Type 
-------------------------- ----------- ----------- -------------------- -------
fe80::5231:cff:fe05:79b7   2226934466  2943854172        Ethernet2(14)  normal 
fe80::5284:b8ff:fe48:4dcb   711531338  1797929343        Ethernet1(13)  normal 

           LastUp       LastDown            LastDiag    State
-------------------- -------------- ------------------- -----
   05/29/24 07:24             NA       No Diagnostic       Up
   05/29/24 07:24             NA       No Diagnostic       Up
```

<!-- TOC --><a name="-loopback"></a>
### Верификация связности между Loopback'ами

<!-- TOC --><a name="spine-1"></a>
#### Spine-1
По очереди сделаем ping на три Leaf'а (loopback Spine-2 недостижим из-за eBGP loop prevention).

ping leaf-1
```
Spine-1#ping 10.1.1.1
PING 10.1.1.1 (10.1.1.1) 72(100) bytes of data.
80 bytes from 10.1.1.1: icmp_seq=1 ttl=65 time=10.1 ms
80 bytes from 10.1.1.1: icmp_seq=2 ttl=65 time=3.38 ms
80 bytes from 10.1.1.1: icmp_seq=3 ttl=65 time=3.45 ms
80 bytes from 10.1.1.1: icmp_seq=4 ttl=65 time=3.64 ms
80 bytes from 10.1.1.1: icmp_seq=5 ttl=65 time=3.70 ms

--- 10.1.1.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 36ms
rtt min/avg/max/mdev = 3.386/4.860/10.109/2.627 ms, ipg/ewma 9.094/7.402 ms
```

ping leaf-2
```
Spine-1#ping 10.1.1.2
PING 10.1.1.2 (10.1.1.2) 72(100) bytes of data.
80 bytes from 10.1.1.2: icmp_seq=1 ttl=65 time=7.29 ms
80 bytes from 10.1.1.2: icmp_seq=2 ttl=65 time=3.58 ms
80 bytes from 10.1.1.2: icmp_seq=3 ttl=65 time=3.87 ms
80 bytes from 10.1.1.2: icmp_seq=4 ttl=65 time=3.32 ms
80 bytes from 10.1.1.2: icmp_seq=5 ttl=65 time=3.50 ms

--- 10.1.1.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 26ms
rtt min/avg/max/mdev = 3.328/4.318/7.290/1.497 ms, ipg/ewma 6.607/5.747 ms
```

ping leaf-3
```
Spine-1#ping 10.1.1.3
PING 10.1.1.3 (10.1.1.3) 72(100) bytes of data.
80 bytes from 10.1.1.3: icmp_seq=1 ttl=65 time=7.62 ms
80 bytes from 10.1.1.3: icmp_seq=2 ttl=65 time=3.49 ms
80 bytes from 10.1.1.3: icmp_seq=3 ttl=65 time=3.77 ms
80 bytes from 10.1.1.3: icmp_seq=4 ttl=65 time=3.53 ms
80 bytes from 10.1.1.3: icmp_seq=5 ttl=65 time=3.43 ms

--- 10.1.1.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 28ms
rtt min/avg/max/mdev = 3.436/4.371/7.623/1.631 ms, ipg/ewma 7.001/5.938 ms
```

Связность есть.

<!-- TOC --><a name="spine-2"></a>
#### Spine-2
По очереди сделаем ping на три Leaf'а (loopback Spine-2 недостижим из-за eBGP loop prevention).

ping leaf-1
```
Spine-2#ping 10.1.1.1
PING 10.1.1.1 (10.1.1.1) 72(100) bytes of data.
80 bytes from 10.1.1.1: icmp_seq=1 ttl=65 time=8.47 ms
80 bytes from 10.1.1.1: icmp_seq=2 ttl=65 time=3.75 ms
80 bytes from 10.1.1.1: icmp_seq=3 ttl=65 time=3.82 ms
80 bytes from 10.1.1.1: icmp_seq=4 ttl=65 time=3.41 ms
80 bytes from 10.1.1.1: icmp_seq=5 ttl=65 time=3.43 ms

--- 10.1.1.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 30ms
rtt min/avg/max/mdev = 3.415/4.581/8.475/1.954 ms, ipg/ewma 7.673/6.451 ms
```

ping leaf-2
```
Spine-2#ping 10.1.1.2
PING 10.1.1.2 (10.1.1.2) 72(100) bytes of data.
80 bytes from 10.1.1.2: icmp_seq=1 ttl=65 time=6.70 ms
80 bytes from 10.1.1.2: icmp_seq=2 ttl=65 time=3.44 ms
80 bytes from 10.1.1.2: icmp_seq=3 ttl=65 time=3.59 ms
80 bytes from 10.1.1.2: icmp_seq=4 ttl=65 time=4.58 ms
80 bytes from 10.1.1.2: icmp_seq=5 ttl=65 time=3.71 ms

--- 10.1.1.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 25ms
rtt min/avg/max/mdev = 3.443/4.406/6.704/1.215 ms, ipg/ewma 6.250/5.526 ms
```

ping leaf-3
```
Spine-2#ping 10.1.1.3
PING 10.1.1.3 (10.1.1.3) 72(100) bytes of data.
80 bytes from 10.1.1.3: icmp_seq=1 ttl=65 time=4.85 ms
80 bytes from 10.1.1.3: icmp_seq=2 ttl=65 time=3.42 ms
80 bytes from 10.1.1.3: icmp_seq=3 ttl=65 time=3.74 ms
80 bytes from 10.1.1.3: icmp_seq=4 ttl=65 time=3.55 ms
80 bytes from 10.1.1.3: icmp_seq=5 ttl=65 time=3.39 ms

--- 10.1.1.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 19ms
rtt min/avg/max/mdev = 3.399/3.793/4.852/0.545 ms, ipg/ewma 4.786/4.302 ms
```

Связность есть.

<!-- TOC --><a name="leaf-1"></a>
#### Leaf-1
По очереди сделаем ping на оба Spine и два остальных Leaf.

ping spine-1
```
Leaf-1#ping 10.1.0.1
PING 10.1.0.1 (10.1.0.1) 72(100) bytes of data.
80 bytes from 10.1.0.1: icmp_seq=1 ttl=65 time=4.85 ms
80 bytes from 10.1.0.1: icmp_seq=2 ttl=65 time=3.72 ms
80 bytes from 10.1.0.1: icmp_seq=3 ttl=65 time=3.42 ms
80 bytes from 10.1.0.1: icmp_seq=4 ttl=65 time=3.67 ms
80 bytes from 10.1.0.1: icmp_seq=5 ttl=65 time=3.45 ms

--- 10.1.0.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 19ms
rtt min/avg/max/mdev = 3.429/3.826/4.851/0.530 ms, ipg/ewma 4.769/4.317 ms
```

ping spine-2
```
Leaf-1#ping 10.1.0.2
PING 10.1.0.2 (10.1.0.2) 72(100) bytes of data.
80 bytes from 10.1.0.2: icmp_seq=1 ttl=65 time=7.37 ms
80 bytes from 10.1.0.2: icmp_seq=2 ttl=65 time=3.53 ms
80 bytes from 10.1.0.2: icmp_seq=3 ttl=65 time=3.67 ms
80 bytes from 10.1.0.2: icmp_seq=4 ttl=65 time=3.69 ms
80 bytes from 10.1.0.2: icmp_seq=5 ttl=65 time=3.59 ms

--- 10.1.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 26ms
rtt min/avg/max/mdev = 3.537/4.376/7.375/1.501 ms, ipg/ewma 6.648/5.825 ms
```

ping leaf-2
```
Leaf-1#ping 10.1.1.2
PING 10.1.1.2 (10.1.1.2) 72(100) bytes of data.
80 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=12.4 ms
80 bytes from 10.1.1.2: icmp_seq=2 ttl=64 time=7.27 ms
80 bytes from 10.1.1.2: icmp_seq=3 ttl=64 time=7.26 ms
80 bytes from 10.1.1.2: icmp_seq=4 ttl=64 time=7.65 ms
80 bytes from 10.1.1.2: icmp_seq=5 ttl=64 time=8.04 ms

--- 10.1.1.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 49ms
rtt min/avg/max/mdev = 7.264/8.544/12.483/1.990 ms, pipe 2, ipg/ewma 12.433/10.464 ms
```

ping leaf-3
```
Leaf-1#ping 10.1.1.3
PING 10.1.1.3 (10.1.1.3) 72(100) bytes of data.
80 bytes from 10.1.1.3: icmp_seq=1 ttl=64 time=10.5 ms
80 bytes from 10.1.1.3: icmp_seq=2 ttl=64 time=7.96 ms
80 bytes from 10.1.1.3: icmp_seq=3 ttl=64 time=8.11 ms
80 bytes from 10.1.1.3: icmp_seq=4 ttl=64 time=7.26 ms
80 bytes from 10.1.1.3: icmp_seq=5 ttl=64 time=6.92 ms

--- 10.1.1.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 41ms
rtt min/avg/max/mdev = 6.927/8.162/10.544/1.270 ms, ipg/ewma 10.250/9.284 ms
```

Связность есть.

<!-- TOC --><a name="leaf-2"></a>
#### Leaf-2
По очереди сделаем ping на оба Spine и два остальных Leaf.

ping spine-1
```
Leaf-2#ping 10.1.0.1
PING 10.1.0.1 (10.1.0.1) 72(100) bytes of data.
80 bytes from 10.1.0.1: icmp_seq=1 ttl=65 time=7.18 ms
80 bytes from 10.1.0.1: icmp_seq=2 ttl=65 time=3.82 ms
80 bytes from 10.1.0.1: icmp_seq=3 ttl=65 time=3.62 ms
80 bytes from 10.1.0.1: icmp_seq=4 ttl=65 time=3.37 ms
80 bytes from 10.1.0.1: icmp_seq=5 ttl=65 time=3.39 ms

--- 10.1.0.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 26ms
rtt min/avg/max/mdev = 3.379/4.279/7.181/1.462 ms, ipg/ewma 6.634/5.670 ms
```

ping spine-2
```
Leaf-2#ping 10.1.0.2
PING 10.1.0.2 (10.1.0.2) 72(100) bytes of data.
80 bytes from 10.1.0.2: icmp_seq=1 ttl=65 time=5.63 ms
80 bytes from 10.1.0.2: icmp_seq=2 ttl=65 time=3.82 ms
80 bytes from 10.1.0.2: icmp_seq=3 ttl=65 time=3.51 ms
80 bytes from 10.1.0.2: icmp_seq=4 ttl=65 time=3.49 ms
80 bytes from 10.1.0.2: icmp_seq=5 ttl=65 time=4.47 ms

--- 10.1.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 21ms
rtt min/avg/max/mdev = 3.496/4.187/5.632/0.807 ms, ipg/ewma 5.250/4.899 ms
```

ping leaf-1
```
Leaf-2#ping 10.1.1.1
PING 10.1.1.1 (10.1.1.1) 72(100) bytes of data.
80 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=12.2 ms
80 bytes from 10.1.1.1: icmp_seq=2 ttl=64 time=7.43 ms
80 bytes from 10.1.1.1: icmp_seq=3 ttl=64 time=7.14 ms
80 bytes from 10.1.1.1: icmp_seq=4 ttl=64 time=7.34 ms
80 bytes from 10.1.1.1: icmp_seq=5 ttl=64 time=8.47 ms

--- 10.1.1.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 44ms
rtt min/avg/max/mdev = 7.146/8.519/12.203/1.900 ms, pipe 2, ipg/ewma 11.096/10.321 ms
```

ping leaf-3
```
Leaf-2#ping 10.1.1.3
PING 10.1.1.3 (10.1.1.3) 72(100) bytes of data.
80 bytes from 10.1.1.3: icmp_seq=1 ttl=64 time=9.67 ms
80 bytes from 10.1.1.3: icmp_seq=2 ttl=64 time=7.63 ms
80 bytes from 10.1.1.3: icmp_seq=3 ttl=64 time=9.09 ms
80 bytes from 10.1.1.3: icmp_seq=4 ttl=64 time=6.73 ms
80 bytes from 10.1.1.3: icmp_seq=5 ttl=64 time=7.72 ms

--- 10.1.1.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 37ms
rtt min/avg/max/mdev = 6.734/8.172/9.671/1.069 ms, ipg/ewma 9.310/8.881 ms
```

Связность есть.

<!-- TOC --><a name="leaf-3"></a>
#### Leaf-3
По очереди сделаем ping на оба Spine и два остальных Leaf.

ping spine-1
```
Leaf-3#ping 10.1.0.1
PING 10.1.0.1 (10.1.0.1) 72(100) bytes of data.
80 bytes from 10.1.0.1: icmp_seq=1 ttl=65 time=4.39 ms
80 bytes from 10.1.0.1: icmp_seq=2 ttl=65 time=3.69 ms
80 bytes from 10.1.0.1: icmp_seq=3 ttl=65 time=3.53 ms
80 bytes from 10.1.0.1: icmp_seq=4 ttl=65 time=3.63 ms
80 bytes from 10.1.0.1: icmp_seq=5 ttl=65 time=3.70 ms

--- 10.1.0.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 16ms
rtt min/avg/max/mdev = 3.532/3.791/4.396/0.315 ms, ipg/ewma 4.146/4.084 ms
```

ping spine-2
```
Leaf-3#ping 10.1.0.2
PING 10.1.0.2 (10.1.0.2) 72(100) bytes of data.
80 bytes from 10.1.0.2: icmp_seq=1 ttl=65 time=5.82 ms
80 bytes from 10.1.0.2: icmp_seq=2 ttl=65 time=4.24 ms
80 bytes from 10.1.0.2: icmp_seq=3 ttl=65 time=3.60 ms
80 bytes from 10.1.0.2: icmp_seq=4 ttl=65 time=3.52 ms
80 bytes from 10.1.0.2: icmp_seq=5 ttl=65 time=3.27 ms

--- 10.1.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 22ms
rtt min/avg/max/mdev = 3.277/4.094/5.826/0.922 ms, ipg/ewma 5.516/4.910 ms
```

ping leaf-1
```
Leaf-3#ping 10.1.1.1
PING 10.1.1.1 (10.1.1.1) 72(100) bytes of data.
80 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=8.26 ms
80 bytes from 10.1.1.1: icmp_seq=2 ttl=64 time=8.09 ms
80 bytes from 10.1.1.1: icmp_seq=3 ttl=64 time=7.39 ms
80 bytes from 10.1.1.1: icmp_seq=4 ttl=64 time=6.84 ms
80 bytes from 10.1.1.1: icmp_seq=5 ttl=64 time=7.21 ms

--- 10.1.1.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 32ms
rtt min/avg/max/mdev = 6.841/7.561/8.260/0.547 ms, ipg/ewma 8.197/7.878 ms
```

ping leaf-2
```
Leaf-3#ping 10.1.1.2
PING 10.1.1.2 (10.1.1.2) 72(100) bytes of data.
80 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=9.76 ms
80 bytes from 10.1.1.2: icmp_seq=2 ttl=64 time=7.13 ms
80 bytes from 10.1.1.2: icmp_seq=3 ttl=64 time=7.17 ms
80 bytes from 10.1.1.2: icmp_seq=4 ttl=64 time=7.58 ms
80 bytes from 10.1.1.2: icmp_seq=5 ttl=64 time=7.78 ms

--- 10.1.1.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 37ms
rtt min/avg/max/mdev = 7.137/7.890/9.767/0.969 ms, ipg/ewma 9.250/8.812 ms
```
Связность есть.

<!-- TOC --><a name="itogo"></a>
## Итого
Связность между loopback'ами всех устройств (кроме Spine<>Spine) достигнута. Задание выполнено.