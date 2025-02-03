#Criar um microserviço serveless para validação de cpf

##Execute o seguinte comando para instalar o Azure Functions Core Tools:
npm install -g azure-functions-core-tools@4 --unsafe-perm true

##Criar uma pasta e em seguida acessar a pasta
mkdir cpfValidatorFunction

cd CpfValidatorFunction

##Execute o comando para criar um novo projeto de Azure Function
func init CpfValidatorFunction --worker-runtime dotnet

##Adicione uma nova função HTTP
func new --name ValidateCpf --template "HTTP trigger"

#Inicie a função
func start

#Implementar a Lógica de Validação de CPF
```
using System;
using System.IO;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;

public static class ValidateCpf
{
    [FunctionName("ValidateCpf")]
    public static IActionResult Run(
        [HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)] HttpRequest req,
        ILogger log)
    {
        log.LogInformation("Iniciando a validação do CPF.");

        string cpf = req.Query["cpf"];

        if (string.IsNullOrEmpty(cpf))
        {
            return new BadRequestObjectResult("Please pass a CPF on the query string");
        }

        bool isValid = IsCpfValid(cpf);

        return new OkObjectResult(new { cpf = cpf, isValid = isValid });
    }

    private static bool IsCpfValid(string cpf)
    {
        // Remove non-numeric characters
        cpf = new string(cpf.Where(char.IsDigit).ToArray());

        // CPF must have 11 digits
        if (cpf.Length != 11)
            return false;

        // Check for known invalid CPFs
        if (cpf.Distinct().Count() == 1)
            return false;

        // Calculate first verification digit
        int sum = 0;
        for (int i = 0; i < 9; i++)
            sum += int.Parse(cpf[i].ToString()) * (10 - i);
        int remainder = sum % 11;
        int digit1 = remainder < 2 ? 0 : 11 - remainder;

        // Calculate second verification digit
        sum = 0;
        for (int i = 0; i < 10; i++)
            sum += int.Parse(cpf[i].ToString()) * (11 - i);
        remainder = sum % 11;
        int digit2 = remainder < 2 ? 0 : 11 - remainder;

        // Verify if the calculated digits match the provided ones
        return cpf[9].ToString() == digit1.ToString() && cpf[10].ToString() == digit2.ToString();
    }
}
```
#Publicar a Função no Azure

##Faça login no Azure:
az login

##Crie um grupo de recursos:
az group create --name CpfValidatorResourceGroup --location eastus

##Crie uma conta de armazenamento:
az storage account create --name cpvalidatorstorage --location eastus --resource-group CpfValidatorResourceGroup --sku Standard_LRS

##Crie um aplicativo de funções:
az functionapp create --resource-group CpfValidatorResourceGroup --consumption-plan-location eastus --runtime dotnet --functions-version 4 --name CpfValidatorApp --storage-account cpvalidatorstorage

##Publique a função:
func azure functionapp publish CpfValidatorApp

##Testar a Função no Azure
#Após a publicação, a URL da função será exibida no terminal. Use essa URL para testar a função no Azure.
https://<your-function-app-name>.azurewebsites.net/api/ValidateCpf?cpf=12345678909
