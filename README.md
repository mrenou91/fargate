
resource "aws_ecs_task_definition" "springboot" {
  family                   = "springboot-task"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "512"
  memory                   = "1024"
  execution_role_arn       = aws_iam_role.ecs_execution_role.arn
  task_role_arn            = aws_iam_role.ecs_task_role.arn

  container_definitions = jsonencode([
    # -----------------------------------------------------
    # INIT CONTAINER
    # -----------------------------------------------------
    {
      name      = "init-config"
      image     = "busybox:latest"
      essential = false
      command   = ["/bin/sh", "-c", "echo 'Init running...'; sleep 5"]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.app_logs.name
          awslogs-region        = var.region
          awslogs-stream-prefix = "init"
        }
      }
    },

    # -----------------------------------------------------
    # MAIN SPRING BOOT CONTAINER
    # -----------------------------------------------------
    {
      name      = "springboot-app"
      image     = var.image   # e.g. harbor.example.com/myteam/myapp:latest
      essential = true

      dependsOn = [
        {
          containerName = "init-config"
          condition     = "SUCCESS"
        }
      ]

      portMappings = [
        {
          containerPort = 8080
          protocol      = "tcp"
        }
      ]

      # 20 environment variables (example)
      environment = [
        { name = "ENV_VAR_1",  value = "value1"  },
        { name = "ENV_VAR_2",  value = "value2"  },
        { name = "ENV_VAR_3",  value = "value3"  },
        { name = "ENV_VAR_4",  value = "value4"  },
        { name = "ENV_VAR_5",  value = "value5"  },
        { name = "ENV_VAR_6",  value = "value6"  },
        { name = "ENV_VAR_7",  value = "value7"  },
        { name = "ENV_VAR_8",  value = "value8"  },
        { name = "ENV_VAR_9",  value = "value9"  },
        { name = "ENV_VAR_10", value = "value10" },
        { name = "ENV_VAR_11", value = "value11" },
        { name = "ENV_VAR_12", value = "value12" },
        { name = "ENV_VAR_13", value = "value13" },
        { name = "ENV_VAR_14", value = "value14" },
        { name = "ENV_VAR_15", value = "value15" },
        { name = "ENV_VAR_16", value = "value16" },
        { name = "ENV_VAR_17", value = "value17" },
        { name = "ENV_VAR_18", value = "value18" },
        { name = "ENV_VAR_19", value = "value19" },
        { name = "ENV_VAR_20", value = "value20" }
      ]

      # Secrets from SSM & SecretsManager
      secrets = concat(
        [
          {
            name      = "DB_URL"
            valueFrom = aws_ssm_parameter.db_url.arn
          },
          {
            name      = "CONFIG_JSON"
            valueFrom = aws_ssm_parameter.app_config.arn
          }
        ],
        [
          {
            name      = "API_KEY"
            valueFrom = aws_secretsmanager_secret_version.api_key.arn
          }
        ]
      )

      # Harbor private registry credentials
      repositoryCredentials = {
        credentialsParameter = aws_secretsmanager_secret.harbor_auth.arn
      }

      # Container health check
      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 20
      }

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.app_logs.name
          awslogs-region        = var.region
          awslogs-stream-prefix = "app"
        }
      }
    }
  ])

  runtime_platform {
    operating_system_family = "LINUX"
    cpu_architecture        = "X86_64"
  }
}

