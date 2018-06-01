---
title: OMNIA Data Source Tutorial
keywords: omnia3
summary: "OMNIA Platform 3.0 data source management"
sidebar: omnia3_sidebar
permalink: omnia3_datasourcetutorial.html
folder: omnia3
---

## 1. Introduction

Based on a simple Employee management scenario, this tutorial shows how easily OMNIA can use and combine information from different data sources.

On the first tutorial area (CRUD Operations), we are going to evaluate how to interact with an external data source, by reading and manipulating its data. On the second area (External data sources data on OMNIA), we are going to focus on the use of data source information on OMNIA's entities. 

As our custom data source, we are going to use a free API named [ReqRes](https://reqres.in/), that simulates real time CRUD operations, based on a user management scenario.

Please notice that, since this is only a simulation, no actual data is manipulated (written, updated or removed) on REQRES's system. However, the code shown will be easily convertable to real-world scenarios. 


## 2. Prerequisites

This tutorial assumes that you have created a OMNIA tenant, and are logged in as a user with modeling privileges to this tenant.

If you do not have a tenant yet, please follow the steps of the [Tenant Creation tutorial](http://docs.numbersbelieve.com/omnia3_tenantcreation.html).

## 3. CRUD operations

1. Start by selecting the tenant where you are going to model, and you will be redirected to the modeling area.

    ![Homepage_Dashboard](http://funkyimg.com/i/2DVGv.png)

2. Through the left side menu, create a new Data Source by accessing the option ***Data Sources / Create new*** on the top right side. Set its Name as "ExternalAPI", Behaviour Runtime as Internal and its Data Access Runtime as External.

    ![Modeler_Create_DataSource](https://raw.githubusercontent.com/numbersbelieve/omnia3/master/docs/tutorialPics/modelingTutorial/Modeler-Create-DataSource.PNG)
    
3. Create a new Agent with name "Employee", and set it as using the external data source "ExternalAPI" that you created earlier.

    ![Modeler_Create_DataSource](https://raw.githubusercontent.com/numbersbelieve/omnia3/master/docs/tutorialPics/modelingTutorial/Modeler-Create-Agent-Employee.PNG)

4. On Agent Employee, navigate to tab "Data Behaviours", and define a behaviour to be executed on "Create". This behaviour will be used to perform a POST request to the external Application when we create an instance of the Employee on the OMNIA platform. Copy and paste the following code:

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

5. On "Data Behaviours" of Agent Employee, define a behaviour, to be executed on "Delete" (when a Employee is deleted on OMNIA). Copy and paste the following code:


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

6. Create a new Data Behaviour for the operation "Read", so that data is retrieved when a Employee is edited on OMNIA. Copy and paste the following code:

    ````
   {% raw %}
    var client = new System.Net.Http.HttpClient();
    string apiEndpoint = $"https://reqres.in/api/users/{identifier}";

    var requestResult = client.GetAsync(apiEndpoint).GetAwaiter().GetResult();

    string responseBody = requestResult.Content.ReadAsStringAsync().Result;
    if (!requestResult.IsSuccessStatusCode)
      throw new Exception("Error on creating contact: " + responseBody);
      
    var response = JsonConvert.DeserializeObject<Dictionary<string, object>>(responseBody);
    var responseData = JsonConvert.DeserializeObject<Dictionary<string, object>>(response["data"].ToString());

    EmployeeDto employeeResponse = new EmployeeDto();
    employeeResponse._code = responseData["id"].ToString();
    employeeResponse._name = $"{responseData["first_name"].ToString()} {responseData["last_name"].ToString()}";

    return employeeResponse;
    {% endraw %}
    ````

7. Create a new Data Behaviour for the operation "ReadList", so that data is retrieved when a list of Employees is requested. Copy and paste the following code:

    ````
  {% raw %}
    var client = new System.Net.Http.HttpClient();
    string apiEndpoint = $"https://reqres.in/api/users?page={page}";

    var requestResult = client.GetAsync(apiEndpoint).GetAwaiter().GetResult();

    string responseBody = requestResult.Content.ReadAsStringAsync().Result;
    if (!requestResult.IsSuccessStatusCode)
      throw new Exception("Error on creating contact: " + responseBody);
      
    var response = JsonConvert.DeserializeObject<Dictionary<string, object>>(responseBody);
    var responseData = JsonConvert.DeserializeObject<List<Dictionary<string, object>>>(response["data"].ToString());

    List<IDictionary<string, object>> employeesList = new List<IDictionary<string, object>>();

    foreach (var employee in responseData)
    {
      var line = new Dictionary<string, object>()
      {{"_code", employee["id"]}, {"_name", employee["first_name"] + " " + employee["last_name"]}};
      employeesList.Add(line);
    }

    return (responseData.Count, employeesList);
  {% endraw %}
    ````

	NOTE: in this scenario, we are ignoring the query sent by the user when obtaining the list. In real world scenarios, you will want to change the query to the external system and/or the returned response, according to the parameters sent by the user.
	
8. Create a new Data Behaviour for the operation "Update", so that data is retrieved when an Employee is updated on OMNIA (i.e., edited and saved). Copy and paste the following code:

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

9. Perform a new Build (by accessing the option ***Versioning / Builds / Create new***).

10. On Application area, create a new instance of the ExternalAPI data source, with code "ReqRes".

    ![Application-Create-DataSource](https://raw.githubusercontent.com/numbersbelieve/omnia3/master/docs/tutorialPics/modelingTutorial/Application-Create-DataSource.PNG)
    
11. On left side menu, navigate to Configurations / Employee, and check that the list is filled with data retrieved from the external data source.

    ![Application_List_DataSource](https://raw.githubusercontent.com/numbersbelieve/omnia3/master/docs/tutorialPics/modelingTutorial/Application-List-External-DataSource.PNG)
    
    
## 4. Crossing the external data sources with OMNIA

1. Go to the modeling area.

2. Create a new GenericEntity with name "Department".

3. Add a new attribute by clicking on button Add new. Set its Code as DataSource, Type as Data Sources / ExternalAPI.

4. Add another attribute that represents the Employee. Set its Code as Employee, Type as Agent / Employee, and ExternalAPI on Uses data source from attribute.

5. Perform a new Build (by accessing the option ***Versioning / Builds / Create new***).

6. On Application area, create a new instance of the Department, and check that, after identifying the data source, Employees from that data source are now available for selection.