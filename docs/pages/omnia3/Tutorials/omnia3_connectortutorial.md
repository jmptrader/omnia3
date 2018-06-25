---
title: OMNIA Connector Tutorial
keywords: omnia3
summary: "OMNIA Platform 3.0 CSV File as Data Source"
sidebar: omnia3_sidebar
permalink: omnia3_connectortutorial.html
folder: omnia3
---

## 1. Introduction

Based on a simple Contact management scenario, this tutorial shows a case of how easily OMNIA can use information from an external data source, using the Omnia connector to access data located on-premises. 

This tutorial is an advanced implementation of the [data sources tutorial](omnia3_datasourcetutorial.html) in order to understand how data sources work, please read [this section of the documentation](omnia3_modeler_datasources.html).

On the CRUD Operations tutorial area, we are going to evaluate how to interact with an external data source, by reading and manipulating its data.

As our custom data source, we are going to use a CSV file with a list of Contacts.

## 2. Prerequisites

This tutorial assumes that you have created a OMNIA tenant ([click here to see how](omnia3_tenantcreation.html)), and are logged in as a user with modeling privileges to this tenant. You must also have access to the management area to manage the connectors.


## 3. Create a new connector

1. Start by accessing the management area, by clicking the option "Go to Tenants management".

2. Through the left side menu, create a new connector by accessing the option ***Connectors / Add new***. Set its Code and Name as "TutorialConnector"

3. Select the connector, and a modal with connector data should be shown.

4. Now we are going to grant the connector access privileges for the tenant. Access the option ***Security / Roles***, and select Administration role for the tenant (the tenant code with prefix "Administration")

5. Click the button ***Add new*** to grant the connector user access to the tenant. The user can be retrieved on step 3, property "Client Email"

6. Now use these configurations to configure a connector in the machine with the Primavera ERP, following the [installation guide](omnia3_connector_install.html) and [configuration guide](omnia3_connector_configuration.html).

7. Start the configured connector.

## 4. CRUD operations

1. Access Omnia homepage, select the tenant where you are going to model and you will be redirected to the modeling area.

2. Through the left side menu, create a new Data Source by accessing the option ***Data Sources / Add new*** on the top right side. Set its Name as "*Primavera*", Behaviour Runtime and Data Access Runtime as *"External"*.

    ![Modeler create DataSource](/images/tutorials/primaveraconnector/add-new-datasource.png)

3. Create a new Agent with name *"Contact"*, and set it as using the external data source *"Primavera"* that you created earlier.

    ![Modeler create Agent](/images/tutorials/primaveraconnector/add-new-agent.png)


5. Navigate to tab *"[Data Behaviours](https://docs.numbersbelieve.com/omnia3_modeler_datasources.html)"*, and define a behaviour to be executed on *"ReadList"*. This behaviour will be used for Query and List requests for this entity.

    Remember to **change** the **```platConfig.Utilizador```** and **```platConfig.PwdUtilizador```** fields to your actual username and password.

    Copy and paste the following code:
    ```C#
    List<IDictionary<string, object>> listData = new List<IDictionary<string, object>>();
    
    int numberOfRecords = 0;
    using (var reader = new System.IO.StreamReader(@"C:\Users\NB\Desktop\Contacts.csv"))
    {
    	while (!reader.EndOfStream)
        {
    		var line = reader.ReadLine();
            var values = line.Split(';');
            Dictionary<string, object> contactData = new Dictionary<string, object>();
            if (values.Length > 1)
            {
    			contactData.Add("_code", values[0]);
                contactData.Add("_name", values[1]);
                numberOfRecords++;
                listData.Add(contactData);
    		}
    	}
    }
    
    return (numberOfRecords, listData);
    ```

6. Create a new Data Behaviour for the operation *"Read"*, so that data is retrieved when an Employee is edited on OMNIA.

    Remember to **change** the **```platConfig.Utilizador```** and **```platConfig.PwdUtilizador```** fields to your actual username and password.

    Copy and paste the following code:

    ```C#
    EmployeeDto dto = new EmployeeDto();
    StdBSConfApl platConfig = new StdBSConfApl();

    platConfig.AbvtApl = "ERP";
    platConfig.Instancia = "default";
    platConfig.Utilizador = "USER";
    platConfig.PwdUtilizador = "PASS";
    platConfig.LicVersaoMinima = "09.00";

    Interop.StdPlatBS900.StdPlatBS bsPlat = new Interop.StdPlatBS900.StdPlatBS();

    Interop.StdBE900.StdBETransaccao trans = null;
    bsPlat.AbrePlataformaEmpresa("DEMO", trans, platConfig, Interop.StdBE900.EnumTipoPlataforma.tpEmpresarial, string.Empty);

    Interop.StdBE900.StdBELista queryResults = bsPlat.Registos.Consulta($"SELECT Codigo, Nome, Email, Telefone FROM Funcionarios WHERE Codigo = '{identifier}'");

    if (!queryResults.Vazia())
    {
        dto._code = queryResults.Valor("Codigo").ToString();
        dto._name = queryResults.Valor("Nome").ToString();

    }
    else {
        throw new Exception($"Could not retrieve Employee with code {identifier}");
    }

    bsPlat.FechaPlataformaEmpresa();

    return dto;
    ```

7. On *"Data Behaviours"* of Agent Employee, define a behaviour, to be executed on *"Update"* (when an Employee is updated on OMNIA). 

    Remember to **change** the **```"USER"```** and **```"PASS"```** fields to your actual username and password.

    Copy and paste the following code:

    ```C#
    ErpBS bsERP = new ErpBS();
    bsERP.AbreEmpresaTrabalho(EnumTipoPlataforma.tpEmpresarial, "DEMO", "USER", "PASS");

    RhpBEFuncionario funcionario = bsERP.RecursosHumanos.Funcionarios.Edita(dto._code);

    funcionario.set_Nome(dto._name);

    bsERP.RecursosHumanos.Funcionarios.Actualiza(funcionario);

    bsERP.FechaEmpresaTrabalho();

    return dto;
    ```

8. Perform a new Build (by accessing the option ***Versioning / Builds / Create new***).

9. Go to the Application area.

10. Create a new instance of the Primavera data source, with code *"DEMO"* and with the Code of the Connector that you have created.

11. On left side menu, navigate to *Configurations / Employee*, identify the Primavera data source instance (DEMO) and check that the list is filled with data retrieved from Primavera.

12. Now you can List and Update Employees directly on your on-premise system, providing your connector is correctly configured and running.