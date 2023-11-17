## OpenTelemetry Demo Walkthrough

1. Checkout a version and run docker compose:

```
git checkout 1.16.0
docker compose up -d
```

2. All services should be up, the interfaces are accessible through the proxy (8080):

```
Frontend: http://localhost:8080/
Feature flags: http://localhost:8080/feature
Load Generator: http://localhost:8080/loadgen/
Jaeger: http://localhost:8080/jaeger/ui
Grafana: http://localhost:8080/grafana
OTel Collector: http://localhost:8080/otlp-http
```

Some other services are exported directly (or on other ports), like:

```
Prometheus: http://localhost:9090/
Configs: http://localhost:10000/

```


Scenarios

Loading frontend page:

1. Go to http://localhost:8080/
2. On jeager, span "frontend-web: documentLoad" with http.url = http://localhost:8080/
  * A lot of static resources are also loaded, in "resourceFetch" spans, like:
  http://localhost:8080/_next/static/css/885c0a76d639b07f.css
  http://localhost:8080/_next/static/chunks/webpack-ded590fe8203ebd1.js
  etc
  * Clicking on show catalog should create a documentLoad with http.url = "http://localhost:8080/#hot-products". Again a bunch of resources are loaded, including some API calls:
  http://localhost:8080/api/products?currencyCode=
  http://localhost:8080/api/cart?sessionId=2c0c88e5-1b8a-4cfc-bded-a7dcd1d330f1&currencyCode=
  http://localhost:8080/api/products?currencyCode=USD
  * The API calls are spans called "frontend-web: HTTP GET" with the http.url of the request, so we can see:
  frontend-web (http.url=http://localhost:8080/api/currency?) -> frontend-proxy (http.url=http://localhost:8080/api/currency?) status 304 -> frontend-proxy egress -> frontend GET -> frontend grpc.oteldemo.CurrencyService/GetSupportedCurrencies -> GRPC to currencyservice CurrencyService/GetSupportedCurrencies
  * http://localhost:8080/api/products?currencyCode=USD
  frontend-web (http.url=http://localhost:8080/api/products?currencyCode=USD) -> frontend-proxy ingress -> frontend-proxy egress -> frontend GET -> frontend grpc.oteldemo.ProductCatalogService/ListProducts -> productcatalogservice oteldemo.ProductCatalogService/ListProducts

  
TODO: Show how to integrate Jaeger and Prometheus data in grafana. Dynatrace and others might have a better out of the box experience



Exemplos com Dynatrace, basicamente é um "override" do docker compose e algumas configs, seguir:
https://www.dynatrace.com/news/blog/opentelemetry-demo-application-with-dynatrace/



### Datadog

Free trial 14 days, free tier with

```
Core collection and visualization features

1-day metric retention
Up to 5 hosts
```

Configure exporter with DATADOG_API_KEY

Install agent on host
DD_API_KEY=YOUR_API_KEY DD_SITE="datadoghq.com" DD_APM_INSTRUMENTATION_ENABLED=host bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script_agent7.sh)"

Interesting tabs on interface
* Logs
* Metrics
* Services -> Host map

### New Relic

Perpetual free tier:

```
100 GB/month of data ingest included.
1 full access user (Telemetry Data Platform, Full-Stack Observability, and Applied Intelligence)
Unlimited free basic users (Telemetry Data Platform data exploration, dashboarding, and New Relic One application building. No Full-Stack Observability or Applied Intelligence access)
```


Configure exporter with OTLP endpoint and LICENSE KEY 

Install new relic agent on host:
curl -Ls https://download.newrelic.com/install/newrelic-cli/scripts/install.sh | bash && sudo NEW_RELIC_API_KEY=API_KEY NEW_RELIC_ACCOUNT_ID=YOUR_ACCOUNT /usr/local/bin/newrelic install

echo "license_key: YOUR_LICENSE" | sudo tee -a /etc/newrelic-infra.yml
curl -fsSL https://download.newrelic.com/infrastructure_agent/gpg/newrelic-infra.gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/newrelic-infra.gpg
echo "deb https://download.newrelic.com/infrastructure_agent/linux/apt/ jammy main" | sudo tee -a /etc/apt/sources.list.d/newrelic-infra.list

### Demo Scenarios

1. Normal flow, show architecture in each system and compare with the drawing on the demo page

Datadog: Service Mgmt -> Service Catalog (Map)

Then show golden signals and the loadgen

* Latency
* Traffic
* Errors
* Saturation

2. Product Catalog Failure

GRPC status codes: https://grpc.github.io/grpc/core/md_doc_statuscodes.html

No Jaeger, buscar pelo serviço de frontend e tag error=true (que começam tem link com o loadgenerator)
O erro vai até o productcatalogservice

Indiretamente falha recomendação pro produto "frontend-proxy:8080/api/recommendations?productIds=L9ECAV7KIM"

No NewRelic a parte interessante fica nos "APM & Services", vai dar pra ver o "error rate" alto pra vários serviços
Summary contém o error rate alto, pode ir na seção de Errors
Lá é pra ter 4 grupos (GET, HTTP GET, POST, HTTP POST)
Procurar uma "occurrence" que tem trace e olhar atributos, como o "http.url"
Ao explorar o trace deve dar pra ver o mapa de serviços e quem falhou (no caso o productcatalog service), olhar os "rpc.*" e o "app.product.id"

No datadog ir pro Service Catalog e deve dar pra ver o error rate alto também na aba de performance
Na aba de reliability tem um issue com o frontend, dá pra seguir até o "Error tracking"
Dá pra ver que os erros são dos paths "api/products" e "api/recommendations"
Clicando neles dá pra ver os "related errors" e aí mostra os traces

3. 