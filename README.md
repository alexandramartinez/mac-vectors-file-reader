# MuleSoft AI Chain (MAC) Vectors + Einstein + Chroma

Simple Mule application making use of the MAC (MuleSoft AI Chain) project to read documents from a given URL and parse them to a JSON String. Using Einstein as language model and Chroma as the vectors database.

## 1 - Salesforce Einstein (Model)

- Make sure you have access to Einstein in Salesforce. Retrieve your organization's URL. For example, `https://mulesoft-dev.develop.lightning.force.com/`
- Go to **Setup** > **App Manager** > **New Connected App** > **Create a Connected App**.

    - Connected App Name: `MuleSoft app`
    - Contact Email: your email
    - Enable OAuth Settings: ✅
    - Callback URL: `https://login.salesforce.com/services/oauth2/callback`
    - Selected OAuth Scopes: `Access Einstein GPT services (einstein_gpt_api)`
    - Enable Client Credentials Flow: ✅

- Press **Save**.
- In the created Connected App detail page, click on **Manage Consumer Details**.
- Copy the consumer key and consumer secret. These are your client id/secret respectively.

## 2 - Chroma (Vector DB)

- Make sure you have [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed.
- Follow the steps [here](https://docs.trychroma.com/deployment/docker) or run:

    ```shell
    docker pull chromadb/chroma
    docker run -p 8000:8000 chromadb/chroma
    ```

## 3 - API Specification

- Go to Design Center in your [Anypoint Platform](https://anypoint.mulesoft.com/) account.
- Create a new RAML 1.0 specification and copy/paste the contents of [this file](/specification/mac-file-reader.raml) or import it directly.
- Publish to Exchange.

## 4 - Implement the Mule app

> [!NOTE]
> You can compare your code to [this](/mule-project/)

- Install the MAC Vectors module with Maven by following [this doc](https://mac-project.ai/docs/ms-vectors/getting-started).
- Add the following Global Elements to your XML file:

    ```xml
    <ms-vectors:config name="MuleSoft_Vectors_Connector_Config" doc:name="MuleSoft Vectors Connector Config" doc:id="14c92cd7-e040-4634-9bb8-04a26539f756">
        <ms-vectors:embedding-model-service>
            <ms-vectors:einstein salesforceOrgUrl="your-sf-org-url" clientId="sf-connectedapp-clientid" clientSecret="sf-connectedapp-clientsecret" />
        </ms-vectors:embedding-model-service>
        <ms-vectors:vector-store>
            <ms-vectors:chroma url="localhost:8000" />
        </ms-vectors:vector-store>
    </ms-vectors:config>
    <file:config name="File_Config" doc:name="File Config" doc:id="4333d39a-0253-4d3b-86f1-c9f168b0920e" />
    <http:request-config name="HTTP_Request_configuration" doc:name="HTTP Request configuration" doc:id="1ac8c8d2-c885-43a8-a30e-96d7a21e789a">
        <http:request-connection host="#[vars.uri.host]" />
    </http:request-config>
    <http:request-config name="HTTPS_Request_configuration" doc:name="HTTP Request configuration" doc:id="88a2acf4-445a-4dbb-afaf-c7cb132cb9ea">
        <http:request-connection host="#[vars.uri.host]" protocol="HTTPS" />
    </http:request-config>
    ```

- Replace the Salesforce credentials with your own.
- Make sure Chroma is indeed running on port `8000` or modify it from the XML.
- Head to the last flow and update it with the next:

    ```xml
    <flow name="post:\reader\url:text\plain:mac-file-reader-config">
        <ee:transform doc:name="Transform Message" doc:id="533db0b0-a059-4170-82a1-c1ab13172465">
            <ee:message />
            <ee:variables>
                <ee:set-variable variableName="uri"><![CDATA[%dw 2.0
    output application/json
    import substringAfter, substringBefore, substringBeforeLast from dw::core::Strings
    var host = (payload substringAfter "://") substringBefore "/"
    var path = ((payload substringAfter host) substringBeforeLast "/") ++ "/"
    ---
    {
        protocol: lower(payload substringBefore ":"),
        host: host,
        path: path,
        fileName: (payload substringAfter path)
    }]]></ee:set-variable>
                <ee:set-variable variableName="workingDir"><![CDATA[%dw 2.0
    output application/json
    ---
    (mule.home default "") ++ "/apps/" ++ (app.name default "")]]></ee:set-variable>
            </ee:variables>
        </ee:transform>
        <choice doc:name="Choice" doc:id="d3934daf-b5c0-4546-94a9-c9dcd8e2ef89">
            <when expression="#[&quot;https&quot; == vars.uri.protocol]">
                <http:request method="GET" doc:name="Request" doc:id="9a407153-2627-4cf9-a776-4078ec1b237a" config-ref="HTTPS_Request_configuration" path="#[vars.uri.path ++ vars.uri.fileName]" />
            </when>
            <otherwise>
                <http:request method="GET" doc:name="Request" doc:id="42240506-b8d5-4417-8e95-842106ea7d53" config-ref="HTTP_Request_configuration" path="#[vars.uri.path ++ vars.uri.fileName]" />
            </otherwise>
        </choice>
        <file:write doc:name="Write" doc:id="ec1fead7-0437-4088-8ffb-fb8cf00845c1" config-ref="File_Config" path="#[vars.workingDir ++ vars.uri.fileName]" />
        <ms-vectors:document-parser doc:name="Document parser" doc:id="0bd74ed0-6448-4ff2-bb5d-4ecc99361534" config-ref="MuleSoft_Vectors_Connector_Config" fileType="any" contextPath="#[vars.uri.fileName]">
            <ms-vectors:storage>
                <ms-vectors:local workingDirecotry="#[vars.workingDir]" />
            </ms-vectors:storage>
        </ms-vectors:document-parser>
        <ee:transform doc:name="Transform Message" doc:id="2add1957-ebe5-468d-b081-3d6d789f074e">
            <ee:message>
                <ee:set-payload><![CDATA[%dw 2.0
    output text/plain
    ---
    payload.text]]></ee:set-payload>
            </ee:message>
        </ee:transform>
    </flow>
    ```