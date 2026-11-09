# Lab 01: Balanceamento de Carga com Nginx

Aluno: Igor Almenara Carneiro

## 1. Ajuste de Porta (8080)

Foram modificados os arquivos docker-compose.yml e nginx.conf. No primeiro, o mapeamento de portas do serviço loadbalancer foi alterado para 8080:8080. No segundo, a diretiva listen foi configurada para a porta 8080.

A infraestrutura foi iniciada com Docker Compose e o funcionamento confirmado por requisições HTTP.

Saída do teste do Load Balancer:

        Hostname: 1f80f902c1f5
        Hostname: ce0490ec01d2
        Hostname: 8bf89f95239c
        Hostname: 1f80f902c1f5
        Hostname: ce0490ec01d2
        Hostname: 8bf89f95239c
        Hostname: 1f80f902c1f5
        Hostname: ce0490ec01d2
        Hostname: 8bf89f95239c
        Hostname: 1f80f902c1f5

## 2. Teste dos Algoritmos de Balanceamento

### 2.1. Round Robin com Peso (Weighted)

Qual configuração foi aplicada no nginx.conf?

Foi atribuído peso 3 ao primeiro back-end e peso 1 aos demais.

    upstream meus_servidores {
        server backend1:80 weight=3;
        server backend2:80 weight=1;
        server backend3:80 weight=1;
    }

Evidência da distribuição desproporcional (saída do terminal):

        60 Hostname: 1f80f902c1f5
        20 Hostname: 8bf89f95239c
        20 Hostname: ce0490ec01d2

O teste com 100 requisições apresentou distribuição de 60% para o primeiro servidor e 20% para cada um dos outros dois, conforme a proporção dos pesos configurados.

### 2.2. IP Hash (Persistência de Sessão)

Evidência de que o tráfego foi fixado em um único nó:

   20 Hostname: 1f80f902c1f5

Foi configurada a diretiva ip_hash no bloco upstream, utilizando o IP do cliente como critério para selecionar o back-end.

    upstream meus_servidores {
        ip_hash;
        server backend1:80;
        server backend2:80;
        server backend3:80;
    }

### 2.3. Least Connections

Trecho de configuração aplicado:

    upstream meus_servidores {
        least_conn;
        server backend1:80;
        server backend2:80;
        server backend3:80;
    }

Foram realizadas 30 requisições com até 10 execuções simultâneas. Os três servidores receberam a mesma quantidade de requisições:

    10 Hostname: 1f80f902c1f5
    10 Hostname: 8bf89f95239c
    10 Hostname: ce0490ec01d2

## 3. Segurança e Rastreabilidade

### 3.1. Rate Limiting

Evidência do bloqueio por excesso de requisições (comprovando o erro 503):

Foi configurado o limite de uma requisição por segundo para cada endereço IP, utilizando as diretivas limit_req_zone e limit_req.

Ao executar dez requisições consecutivas, o resultado foi:

    1 200
    9 503

Apenas uma requisição foi atendida com sucesso, enquanto as outras nove foram rejeitadas com o código HTTP 503, demonstrando o funcionamento do mecanismo de limitação.

### 3.2. Injeção de Headers de Proxy

Evidência da aplicação back-end recebendo o IP real do cliente:

Foram configurados os cabeçalhos X-Real-IP e X-Forwarded-For para encaminhar informações sobre a origem das requisições aos back-ends.

    Hostname: ce0490ec01d2
    RemoteAddr: 172.18.0.5:37914
    X-Forwarded-For: 172.18.0.1
    X-Real-Ip: 172.18.0.1

A resposta confirma o recebimento dos cabeçalhos pela aplicação. Como a infraestrutura foi executada no GitHub Codespaces, o endereço 172.18.0.1 corresponde à rede interna do Docker.

## 4. Análise: Limitações da versão Open Source

Pergunta: Por que a versão Open Source do Nginx não possui suporte nativo aos algoritmos Least Response Time e Adaptive Routing? Como essas limitações são tratadas no mercado?

Resposta: O Nginx Open Source não oferecia suporte ao algoritmo Least Response Time. A função era exclusiva do Nginx Plus, versão comercial da ferramenta. Contudo, isso mudou na versão 1.31.0 da tecnologia, inaugurada em maio deste ano já com suporte ao Least Response Time.

O Adaptive Routing exige monitoramento e ajustes dinâmicos na distribuição de tráfego, variando conforme o desempenho e a disponibilidade dos servidores. O Nginx Open Source oferece diferentes estratégias de balanceamento, mas a distribuição dinâmica completa requer mecanismos de monitoramento e controle adicionais -- estes, disponíveis no Nginx Plus, como verificações ativas de integridade e reconfiguração dinâmica de servidores. Sendo assim, o método de distribuição dinâmica só está disponível na versão paga.

Como alternativa, o mercado enfrenta essa limitação de três formas:
- Contrata a versão comercial Nginx Plus para ter acesso às ferramentas exclusivas, como verificação ativa de integridade, e API que permite ajuste dinâmico de pesos e servidores disponíveis;
- Combinam o Nginx Open Source com ferramentas externas, como Grafana ou controladores próprios, para monitorar o desempenho de servidores e automatizar alterações;
- Outros proxies e balanceadores: há soluções como Envoy que oferecem recursos de direcionamento adaptativo. Não é exatamente a mesma coisa que o Adaptive Routing, mas é uma solução equivalente.