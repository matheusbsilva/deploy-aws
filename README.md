# Como fazer o deploy no AWS(WIP)

## Deploy no AWS ECS

### Pré-requisitos

1. [Instalar ecs-cli](https://docs.aws.amazon.com/pt_br/AmazonECS/latest/developerguide/ECS_CLI_installation.html)
2. [Instalar aws-cli](https://aws.amazon.com/pt/cli/)  

### Deploy no modo [FARGATE](https://aws.amazon.com/pt/fargate/)

1. Configure o ECS-cli  
  - Configure o profile na **ecs-cli**. [Como obter as access_keys](https://aws.amazon.com/pt/blogs/security/wheres-my-secret-access-key/)
  
  ecs-cli configure profile --access-key AWS_ACCESS_KEY_ID --secret-key AWS_SECRET_ACCESS_KEY --profile-name tutorial  
  - Configure o cluster no qual o deploy será realizado: 
  
  ecs-cli configure --cluster tutorial --region us-east-1 --default-launch-type FARGATE --config-name tutorial
  
2. Crie o cluster e o security group  
  - Suba o cluster com: 
  
  ecs-cli up
  
  - Ao subir o cluster será criada a [VPC](https://aws.amazon.com/pt/vpc/), o comando acima retornará o **id** da **VPC** criada e os **ids** das subnets, o retorno vem no formato:  
  
  VPC created: vpc-XXXXXXXX
  Subnet created: subnet-XXXXXXXXX
  Subnet created: subnet-XXXXXXXXX
 
  - Usando a AWS-Cli crie um grupo de segurança usando o id da VPC obtido no passo anterior, ao criar o grupo o id do mesmo será retornado:
  
  
  # Para criar o grupo de segurança
  aws ec2 create-security-group --group-name "my-sg" --description "My security group" --vpc-id "VPC_ID" 
  

  
  # Retorno com o id do grupo
  {
    "GroupId": "sg-XXXXXXXXXX"
  }
  
  - Usando a AWS-cli adicione uma regra para o grupo de segurança poder acessar a porta 80, usando como parâmetro o id do grupo retornado no passo anterior
  aws ec2 authorize-security-group-ingress --group-id "security_group_id" --protocol tcp --port 80 --cidr 0.0.0.0/0 
  

3. Crie um compose file
  - Crie um docker-compose.yml para sua aplicação  
  
  - Crie um ecs-params.yml para configurar VPC, subnets e security groups. [Mais sobre ecs-params](https://docs.aws.amazon.com/pt_br/AmazonECS/latest/developerguide/cmd-ecs-cli-compose.html#cmd-ecs-cli-compose-ecsparams). O básico para rodar a aplicação é no seguinte formato:  
  
  
  version: 1
  task_definition:
    ecs_network_mode: awsvpc
    task_size:
      mem_limit: 0.5GB
      cpu_limit: 256
    run_params:
      network_configuration:
        awsvpc_configuration:
          subnets:
            - "subnet ID 1"
            - "subnet ID 2"
          security_groups:
            - "security group ID"
          assign_public_ip: ENABLED
 
    
4. Faça o deploy do cluster

	ecs-cli compose --project-name tutorial --file docker-compose.prod.yml --ecs-params ecs-params.yml service up --cluster-config tutorial


Observações:
  - Lauchtype FARGATE só suporta o modo de rede awsvpc, para utilizar o links no docker-compose o tipo de network deve ser bridge;
  - No modo awsvpc do FARGATE é possível comunicar entre os containers através do localhost, pois como os containers não são linkados não existe um DNS com o nome de cada serviço;
  - No modo FARGATE não encontrei uma maneira de persistir fora do container os dados.


