+++
author = "Richelmy Monteiro"
title = "Como utilizar Redis no FLUIG"
date = "2026-05-03"
description = "Chamar funções do Redis para usar de cache nas suas aplicações FLUIG"

+++
As instruções abaixo não contam com suporte oficial da TOTVS e devem ser realizadas por sua conta em risco, em caso de dúvidas entre em contato com um consultor FLUIG de sua confiança.  

Redis é utilizado em larga escapa com aplicações distribuídas na web, servindo de cache em memória RAM para retornar rapidamente informações consultadas com frequência.  
Integrá-lo a seu servidor do FLUIG pode trazer os benefícios:  
- Ganho de performance
- Reduzir a quantidade de chamadas de API de terceiros ou ao ERP do cliente

# Caso de uso
Um programa que recebe o número do CPF e verifica no ERP se este usuário já está cadastrado.  
Cenário atual:  
Cada consulta no banco de dados do ERP leva 5 segundos, é um programa legado e está fora do seu alcance alterar o funcionamento dele, seja mudando as queries SQL, incluir índices ou fazer outro tipo de otimização.  
Um usuário que necessita constantemente verificar se o CPF está cadastrado vai levar o tempo para digitar os números mais o tempo de consulta, repetidamente, todos os dias. Não vou entrar no mérito de considerar o tempo perdido pelo usuário diariamente apenas esperando o sistema retornar se o CPF existe ou não, mas que melhoras de performance tornariam o sistema mais amigável.  

## Servidor compatível com Redis
Você pode utilizar o próprio redis-server ou a alternativa comunitária valkey-server. Valkey está disponível via repositório EPEL no Oracle Linux 10 ou via repositório security do Debian 13. Valkey foi criado como um fork após a companhia responsável pelo Redis alterar a licença de código aberto para uma licença não aberta, é um substituto transparente ao Redis, sem necessitar adaptações adicionais.  
```bash
apt install valkey-server
```
Habilite e inicie o serviço systemd
```bash
systemctl enable valkey-server && systemctl start valkey-server
```
O servidor começará a escutar por padrão na porta 6379 da interface loopback (127.0.0.1 em ipv4), recomendo fortemente que o servidor valkey seja executado no mesmo servidor do FLUIG.  

## Extração de dados
Periodicamente são consultados todos os CPFs no ERP e armazenados no servidor valkey, convém que o ERP possua registro de quando um determinado CPF foi cadastrado, para que a extração consulte apenas a partir da última data consumida, sem puxar a base toda.  
O processo de extração não depende do FLUIG em si, poderia ser feito por um script python ou no ERP em si. Vou presumir que todos os CPFs cadastrados no momento estejam disponíveis no servidor valkey.  

## Chamada dos dados do FLUIG
Jedis, a biblioteca do Java para consultar a API do Redis, já está pré-instalada no FLUIG por padrão, então não é preciso baixar e alterar os módulos da aplicação, apenas usar.  

Exemplo de como o dataset de busca de CPF poderia ser:
```javascript
function defineStructure() {
}
function onSync(lastSyncDate) {
}
function createDataset(fields, constraints, sortFields) {
    var newDataset = DatasetBuilder.newDataset(); 
	newDataset.addColumn("STATUS");
	newDataset.addColumn("VALOR");
    try{
        // Chamada ao módulo do Jedis usando caminho completo
        var Jedis = Packages.redis.clients.jedis.Jedis;
        // Conexão no servidor local na porta padrão
        var jedis = new Jedis("localhost", 6379);

        if(constraints[0].fieldName == 'CONSULTA_CPF' && constraints[0].initialValue != ''){
            var str_cpf = constraints[0].initialValue;
            var r = validaCPF(str_cpf) // Código de validação ausente por ser apenas um exemplo
            if(r){
                id = jedis.get("cpf: "+str_cpf);
                if(id != null){
                    newDataset.addRow(Array(
                        0, // Não houve erro
                        id // Retorna o ID do cliente cadastrado, pode ser recebido pela aplicação apenas para informar que já existe ou buscar detalhes baseado no ID de usuário.
                    ));
                    return newDataset;
                }else{
                    /**
                     * Código para verificar se o CPF existe no ERP
                     * Necessário caso a extração esteja atrasada em relação aos dados da base
                     * e garantir que haja retorno a aplicação mesmo caso o cache falhe
                     */
                    return newDataset
                }
            }else{
                newDataset.addRow(Array(
                    1, // Valor recebido não é um CPF
                    "ds_cpf: CPF não possui formato válido " + str_cpf
                    ));
                return newDataset
            }
        }else{
            newDataset.addRow(Array(
                1, // Erro de constraints
                "Verifique as constraints usadas para chamar este dataset"
            ))
            return newDataset;
        }
    }except(e){
        // Exceção encontrada, possivelmente ao tentar conectar no servidor
        newDataset.addRow(Array(
            2,// status de erro
            "Consulta ao dataset do Redis falhou: "+e.message
        ))
        return newDataset;
    }
}
function onMobileSync(user) {
}
```