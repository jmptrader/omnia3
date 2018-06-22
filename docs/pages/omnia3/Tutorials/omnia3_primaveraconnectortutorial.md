---
title: OMNIA Primavera Connector Tutorial
keywords: omnia3
summary: "OMNIA Platform 3.0 ERP Primavera as Data Source"
sidebar: omnia3_sidebar
permalink: omnia3_primaveraconnectortutorial.html
folder: omnia3
---

## 1. Introduction

Based on a simple Employee management scenario, this tutorial shows a real scenario of how easily OMNIA can use information from an external data source, using the Omnia connector to access data located on-premises. 

This tutorial is an advanced implementation of the [data sources tutorial](omnia3_datasourcetutorial.html) In order to understand how data sources work, please read [this section of the documentation](omnia3_modeler_datasources.html).

On the first tutorial area (CRUD Operations), we are going to evaluate how to interact with an external data source, by reading and manipulating its data. On the second area (External data sources data on OMNIA), we are going to focus on the use of data source information on OMNIA's entities. 

As our custom data source, we are going to use a free API named [ReqRes](https://reqres.in/), that simulates real time CRUD operations, based on a user management scenario.

Please notice that, since this is only a simulation, no actual data is manipulated (written, updated or removed) on REQRES's system. However, the code shown will be easily convertable to real-world scenarios. 


## 2. Prerequisites

This tutorial assumes that you have created a OMNIA tenant, and are logged in as a user with modeling privileges to this tenant. You must also have access to the management area to manage the connectors.

If you do not have a tenant yet, please follow the steps of the [Tenant Creation tutorial](http://docs.numbersbelieve.com/omnia3_tenantcreation.html).

This tutorial also requires an access to [Primavera ERP](https://pt.primaverabss.com), on version 9. 

## 3. Create a new connector

1. Start by accessing the management area, by clicking the option "Go to Tenants management".

2. Through the left side menu, create a new connector by accessing the option ***Connectors / Create new***. Set its Code and Name as "TutorialConnector"

3. Select the connector, and a modal with connector data should be shown.

4. Now we are going to grant the connector access privileges for the tenant. Access the option ***Security / Roles***, and select Administration role for the tenant (the tenant code with prefix "Administration")

5. Click on button ***Add new*** to grant the connector user access to tenant. User can be retrieved on step 3, property "Client Email"

## 3. CRUD operations

1. Access Omnia homepage, select the tenant where you are going to model and you will be redirected to the modeling area.

    ![Homepage_Dashboard](http://funkyimg.com/i/2DVGv.png)

2. Through the left side menu, create a new Data Source by accessing the option ***Data Sources / Create new*** on the top right side. Set its Name as "Primavera", Behaviour Runtime and Data Access Runtime as External.

    ![Modeler_Create_DataSource](https://raw.githubusercontent.com/numbersbelieve/omnia3/master/docs/tutorialPics/modelingTutorial/Modeler-Create-DataSource.PNG)
    
3. Create a new Agent with name "Employee", and set it as using the external data source "Primavera" that you created earlier.

    ![Modeler_Create_DataSource](https://raw.githubusercontent.com/numbersbelieve/omnia3/master/docs/tutorialPics/modelingTutorial/Modeler-Create-Agent-Employee.PNG)

4. On Agent Employee, navigate to tab "Data References", and define a reference for Primavera assemblies "Interop.StdPlatBS900.dll" and "Interop.StdBE900.dll"

5. Navigate to tab "Data Behaviours", and define a behaviour to be executed on "Create". This behaviour will be used to perform a POST request to the external Application when we create an instance of the Employee on the OMNIA platform. Copy and paste the following code:

    ````
    {% raw %}
    var client = new System.Net.Http.HttpClient();
    
    string apiEndpoint = $"https://reqres.in/api/users/";

    var body = new
    {
      code = dto._code,
      name = dto._name
    };

    var jsonBody = JsonConvert.SerializeObject(body);
    var httpContent = new System.Net.Http.StringContent(jsonBody, System.Text.Encoding.UTF8, "application/json");

    var requestResult = client.PostAsync(apiEndpoint, httpContent).GetAwaiter().GetResult();

    string responseBody = requestResult.Content.ReadAsStringAsync().Result;

    if (!requestResult.IsSuccessStatusCode)
      throw new Exception("Error on creating contact: " + responseBody);

    var response = JsonConvert.DeserializeObject<Dictionary<string, object>>(responseBody);

    EmployeeDto employeeResponse = new EmployeeDto();
    employeeResponse._code = response["code"].ToString();
    employeeResponse._name = response["name"].ToString();
    return employeeResponse;
      {% endraw %}
    ````

6. On "Data Behaviours" of Agent Employee, define a behaviour, to be executed on "Delete" (when a Employee is deleted on OMNIA). Copy and paste the following code:


    ````
    {% raw %}
    var client = new System.Net.Http.HttpClient();
    
    string apiEndpoint = $"https://reqres.in/api/users/{identifier}";

    var requestResult = client.DeleteAsync(apiEndpoint).GetAwaiter().GetResult();

    string responseBody = requestResult.Content.ReadAsStringAsync().Result;

    if (!requestResult.IsSuccessStatusCode)
      throw new Exception("Error on removing Employee: " + responseBody);

    return true;
    {% endraw %}
    ````

7. Create a new Data Behaviour for the operation "Read", so that data is retrieved when a Employee is edited on OMNIA. Copy and paste the following code:

    ````
   {% raw %}
            EmployeeDto dto = new EmployeeDto();
            StdBSConfApl platConfig = new StdBSConfApl();

            platConfig.AbvtApl = "ERP";
            platConfig.Instancia = "default";
            platConfig.Utilizador = "[PrimaveraUSER]";
            platConfig.PwdUtilizador = "[PrimaveraPWD]";
            platConfig.LicVersaoMinima = "09.00";

            Interop.StdPlatBS900.StdPlatBS bsPlat = new Interop.StdPlatBS900.StdPlatBS();

            Interop.StdBE900.StdBETransaccao trans = null;
            bsPlat.AbrePlataformaEmpresa("[PrimaveraCOMPANY]", trans, platConfig, Interop.StdBE900.EnumTipoPlataforma.tpEmpresarial, string.Empty);

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
    {% endraw %}
    ````

8. Create a new Data Behaviour for the operation "ReadList", so that data is retrieved when a list of Employees is requested. Copy and paste the following code:

    ````
  {% raw %}
            try
            {
                List<IDictionary<string, object>> employeesList = new List<IDictionary<string, object>>();

                StdBSConfApl platConfig = new StdBSConfApl();

                platConfig.AbvtApl = "ERP";
                platConfig.Instancia = "default";
                platConfig.Utilizador = "[PrimaveraUSER]";
                platConfig.PwdUtilizador = "[PrimaveraPWD]";
                platConfig.LicVersaoMinima = "09.00";

                Interop.StdPlatBS900.StdPlatBS bsPlat = new Interop.StdPlatBS900.StdPlatBS();

                Interop.StdBE900.StdBETransaccao trans = null;
                bsPlat.AbrePlataformaEmpresa("[PrimaveraCOMPANY]", trans, platConfig, Interop.StdBE900.EnumTipoPlataforma.tpEmpresarial, string.Empty);

                Interop.StdBE900.StdBELista queryResults = bsPlat.Registos.Consulta($"SELECT Employees.EmployeesCount, Codigo, Nome FROM Funcionarios CROSS JOIN (SELECT Count(*) AS EmployeesCount FROM Funcionarios) AS Employees ORDER BY Codigo OFFSET {(page - 1)*pageSize} ROWS FETCH NEXT {pageSize} ROWS ONLY");

                int numberOfRecords = Convert.ToInt32(queryResults.Valor("EmployeesCount").ToString());
                while (!queryResults.NoFim())
                {

                    var employee = new Dictionary<string, object>() {
                        { "_code", queryResults.Valor("Codigo").ToString()},
                        { "_name", queryResults.Valor("Nome").ToString()}
                    };

                    employeesList.Add(employee);
                    queryResults.Seguinte();
                }
                
                bsPlat.FechaPlataformaEmpresa();
                return (numberOfRecords, employeesList);
            }
            catch (Exception e)
            {
                Console.WriteLine(e.Message);
                throw;
            }
  {% endraw %}
    ````

	NOTE: in this scenario, we are ignoring the query sent by the user when obtaining the list. In real world scenarios, you will want to change the query to the external system and/or the returned response, according to the parameters sent by the user.
	
9. Create a new Data Behaviour for the operation "Update", so that data is retrieved when an Employee is updated on OMNIA (i.e., edited and saved). Copy and paste the following code:

    ````
    {% raw %}
    var client = new System.Net.Http.HttpClient();
    string apiEndpoint = $"https://reqres.in/api/users/{dto._code}";

    var body = new
    {
      code = dto._code,
      name = dto._name
    };
    
    var jsonBody = JsonConvert.SerializeObject(body);

    var httpContent = new System.Net.Http.StringContent(jsonBody, System.Text.Encoding.UTF8, "application/json");

    var requestResult = client.PutAsync(apiEndpoint, httpContent).GetAwaiter().GetResult();
    string responseBody = requestResult.Content.ReadAsStringAsync().Result;
    
    if (!requestResult.IsSuccessStatusCode)
      throw new Exception("Error on creating contact: " + responseBody);

    var response = JsonConvert.DeserializeObject<Dictionary<string, object>>(responseBody);

    EmployeeDto employeeResponse = new EmployeeDto();
    employeeResponse._code = response["code"].ToString();
    employeeResponse._name = response["name"].ToString();

    return employeeResponse;
    {% endraw %}
    ````
 

10. Perform a new Build (by accessing the option ***Versioning / Builds / Create new***).

11. On Application area, create a new instance of the Primavera data source, with code "DEMO".

    ![Application-Create-DataSource](https://raw.githubusercontent.com/numbersbelieve/omnia3/master/docs/tutorialPics/modelingTutorial/Application-Create-DataSource.PNG)
    
12. On left side menu, navigate to Configurations / Employee, identify the Primavera data source instance (DEMO) and check that the list is filled with data retrieved from Primavera.

    ![Application_List_DataSource](https://raw.githubusercontent.com/numbersbelieve/omnia3/master/docs/tutorialPics/modelingTutorial/Application-List-External-DataSource.PNG)
    