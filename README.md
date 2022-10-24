## Enterprise Translation - A foundational use case for customizing Amazon Translate

This post provides an example of how to build a centralized customizable translation solution using Amazon Translate, Amazon API Gateway, AWS Lambda and Amazon DynamoDB. For details about the use-case, please refer to the [blog post](https://aws.amazon.com/blogs/machine-learning/a-foundational-use-case-for-customizing-amazon-translate/).

The solution uses the native features of Amazon Translate, including real-time translation, automatic source language detection (powered by Amazon Comprehend), and custom translation using custom terminology. You can expose these features as one simple <i>/translate</i> API using API Gateway. In order for custom terminology to work, you also need to upload the terminology files to Amazon Translate. Therefore, the API <i>/customterm</i> is exposed to do that.

The solution illustrates two options for translation: a standard translation and a customized translation (using the custom terminology feature). However, you can modify these options as needed to suit your business requirements. Consumers can use these options using [API keys](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-usage-plans.html) of API Gateway. When a translation request is received by the API, it validates (using an [AWS Lambda](http://aws.amazon.com/lambda) authorizer function) whether the provided API key is authorized to perform the type of translation requested. We use an [Amazon DynamoDB](https://aws.amazon.com/dynamodb/) table to store metadata information about consumers, permissions, and API keys.

Three consumer types are defined as part of the solution deployment:

<b>Standard translation persona</b> – Uses the standard translation option, allowing text translation, including automatic language detection

<b>Customized translation persona</b> – Uses the customized translation option, allowing all features of standard translation and also custom translations using a custom terminology file

<b>Admin persona</b> – Supports the customized translation option, allowing the upload of a custom terminology file, but isn’t able to make any other translation API calls

The following diagram illustrates the solution architecture.

<p align="center">
  <img src="Resources/EnterpriseTranslationArchitecture.png" alt="Solution architecture diagram" />
</p>

<h4>For the user translation persona, the process includes the following actions (the blue path in the preceding diagram):</h4>

1a/ The user calls the <i>/translate</i> API and passes the API key in the API header. Optionally, for the customized translation persona, the user can enable custom translation by passing in an optional query string parameter (<i>useCustomTerm</i>).

2/ API Gateway validates the API key.

3/ The Lambda custom authorizer is called to validate the action that the supplied API key is allowed. For instance, a standard translation persona can’t ask for custom translation, or an administrator can’t perform any text translation.

4/ The Lambda authorizer gets the user information from the DynamoDB table and verifies against the API key provided.

5a/ After validation, another Lambda function (Translate) is invoked to call the Amazon Translate API <i>translate_text</i>.

6a/ The translated text is returned in the API response.


<h4>The admin persona can upload a custom terminology file that can be used by the customized translation persona by calling the <i>/customterm API</i>. The workflow steps are follows (the green path in the preceding diagram):</h4>

1b/ The user calls the <i>/customterm</i> API and passes the API key in the API header.

2/ API Gateway validates the API key.

3/ The Lambda custom authorizer is called to validate the action that the supplied API key is allowed. For instance, only an admin persona can upload custom terminology files.

4/ The Lambda authorizer gets the user information from the DynamoDB table and verifies against the API key provided.

5b/ After the API key is validated, another Lambda function (Upload) is invoked to call the Amazon Translate API <i>import_terminology</i>.

6b/ The custom terminology file is uploaded to Amazon Translate with a unique name generated by the Lambda function.

<h3>Deploy the solution with AWS CloudFormation</h3>

Launch the provided CloudFormation template (<i>aws-enterprise-translate.yaml</i>) to deploy the solution in your AWS account. It can be deployed in any AWS region where Amazon Translate is supported.

1/ Open CloudFormation page in console and select <b>Create stack with new resources</b>

2/ Provide the location of CloudFormation template from this repo (either stored locally or in your AWS S3 Bucket)

3/ Choose <b>Next</b>.

4/ For <b>Stack name</b>, enter the name of the CloudFormation stack (for this post, EnterpriseTranslate).

5/ For <b>DDBTableName</b>¸ enter the name of the DynamoDB table (EnterpriseTranslateTable).

6/ For <b>apiGatewayName</b>, enter the API Gateway created by the stack (EnterpriseTranslateAPI).

7/ For <b>apiGatewayStageName</b>, enter the environment name for API Gateway (prod).

8/ Choose <b>Next</b>.

9/ On the review page, select the check boxes to acknowledge the creation of IAM resources.This is required to allow CloudFormation to create a role to grant access to the resources needed by the stack and name the resources in a dynamic way.

10/ Choose <b>Create stack</b>.

The deployment creates the following resources (all prefixed with <i>EntTranslate</i>):

- An API Gateway API with two resources called <i>/customterm</i> and <i>/translate</i>, with three API keys to represent two translation personas and an admin persona

- A DynamoDB table with three items to reflect one consumer with three different roles (three API keys)

- Several Lambda functions (using Python 3.9) as per the architecture diagram

After the resources are deployed into your account on the AWS Cloud, you can test the solution.

<h3>Testing the solution</h3>

<h4>Collect API keys</h4>

Complete the following steps to collect the API keys:

- Navigate to the <b>Outputs</b> tab of the CloudFormation stack and copy the value of the key <i>apiGatewayInvokeURL</i>.To find the API keys created by the solution, look in the DynamoDB table you just created or navigate to the API keys page on the API Gateway console. This post uses the latter approach.

- On the <b>Resources</b> tab of the CloudFormation stack, find the logical ID <i>EntTranslateApi</i> and open the link under the <b>Physical ID</b> column in a new tab.

- On the API Gateway console, choose <b>API Keys</b> in the navigation pane.

- Note the three API keys (standard, customized, admin) generated by the solution. For example, select EntTranslateCus1StandardTierKey and choose <b>Show link</b> against the API key property.

Now you can test the APIs using any open-source tools of your choosing. For this post, we use the [Postman](https://www.postman.com/) API testing tool for illustration purposes only. For details on testing API with Postman, refer to [API development overview](https://learning.postman.com/docs/designing-and-developing-your-api/the-api-workflow/).

<h4>Test 1: Standard translation</h4>

To test the standard translation API, you first create a POST request in Postman.

- Choose <b>Add Request</b> in Postman.

- Set the method type as <b>POST</b>.

- Enter the API Gateway invoke URL.

- Add <i>/translate</i> to the URL endpoint.

- On the <b>Headers</b> tab, add a new header key named <i>x-api-key</i>.

- Enter the standard API key value.

- On the <b>Body</b> tab, enter a JSON body as follows:

{
            "sourceText": "some text to translate",
            "targetLanguage": "fr",
            "sourceLanguage":"en"
}

<i>sourceLanguage</i> is an optional parameter. If you don’t provide it, the system will set it as auto for the automatic detection of the source language.

- Call the API by choosing <b>Send</b> and verify the output.

<p align="center">
  <img src="Resources/ML-10275-image003.png" alt="Postman screen1" />
</p>

The API should run successfully and return the translated text in the <b>Body</b> section of the response object.

<h4>Test 2: Customized translation with custom terminology</h4>

To test the custom term upload functionality, we first create a PUT request in Postman.

- Choose Add Request in Postman.

- Set the method type as PUT.

- Enter the API Gateway invoke URL.

- Add /customterm to the end of the URL.

- On the Headers tab, add a new header key named x-api-key.

- Enter the admin API key value.

- On the Body tab, change the format to binary and upload the custom term CSV file.A sample CSV file is provided under the /Resources folder.

- Call the API by choosing Send and verify the output.

<p align="center">
  <img src="Resources/ML-10275-image005.png" alt="Postman screen1" />
</p>

The API should run successfully with a message in the Body section of the response object saying “Custom term uploaded successfully”

- On the Amazon Translate console, choose Custom Terminology in the navigation pane.
A custom terminology file should have been uploaded and is displayed in the terminology list. The file name syntax is the customer ID from the DynamoDB table for the selected API key followed by string _customterm_1.

<p align="center">
  <img src="Resources/ML-10275-image007.png" alt="Postman screen1" />
</p>

Note that if you didn’t use the admin API key, the system will fail to upload the custom term file.Now you’re ready to perform your custom translation.

- Choose Add Request in Postman.

- Set the method type as POST.

- Enter the API Gateway invoke URL.

- Add /translate to the URL endpoint.

- On the Headers tab, add a new header key named x-api-key.

- Enter the standard API key value.

- On the Body tab, enter a JSON body as follows:

{
            "sourceText": "some text to translate",
            "targetLanguage": "fr",
            "sourceLanguage":"en"
}

- On the Params tab, add a new query string parameter named useCustomTerm with a value of 1.

- Call the API by choosing Send and verify the output.The API should fail with the message “Unauthorized.” This is because you’re trying to call a customized translation feature using a standard persona API key.

- On the Headers tab, enter the customized API key value.

- Run the test again, and it should be able to translate using the custom terminology file.

You will also notice that this time the translated text keeps the word “translate” without translating it (if you used the sample file provided). This is due to the fact that the custom terminology file that was previously uploaded has the word “translate” in it, suggesting that the custom terminology modified the base output from Amazon Translate.

<h4>Test 3: Add additional consumers and tenants</h4>

This solution deployed one consumer (<i>customerA</i>) with three different API keys as part of the CloudFormation stack deployment. You can add additional consumers by creating a new usage plan in API Gateway and associating new API keys to this usage plan. For more details on how to create usage plans and API keys, refer to [Creating and using usage plans with API keys](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-usage-plans.html). You can then add these API keys as additional entries in the DynamoDB table.

<h3>Clean up</h3>

To avoid incurring future charges, clean up the resources you created as part of the CloudFormation stack:

- On the AWS CloudFormation console, navigate to the stack you created.

- Select the stack and choose <b>Delete</b> stack.

Your stack might take some time to be deleted. You can track its progress on the Events tab. When the deletion is complete, the stack status changes from <i>DELETE_IN_PROGRESS</i> to <i>DELETE_COMPLETE</i>. It then disappears from the list.

<h3>Conclusion</h3>

In this post, we demonstrated how easy it is to perform real-time translation, upload custom terminology files, and do custom translation in Amazon Translate using its native APIs, and created a solution to support customization with API Gateway.

You can extend the solution with customizations that are relevant to your business requirements. For instance, you can provide additional functionality like [Active Custom Translation](https://docs.aws.amazon.com/translate/latest/dg/customizing-translations-parallel-data.html) using parallel data via another API key, or create a caching layer to work with this solution to further reduce the cost of translations and serve frequently accessed translations from a cache. You can even introduce API throttling and rate limiting by taking advantage of API Gateway features. The possibilities are endless, and we would love to hear how you take this solution to the next level for your organization.

For more information about Amazon Translate, visit [Amazon Translate resources](https://aws.amazon.com/translate/resources/?amazon-translate.sort-by=item.additionalFields.postDateTime&amazon-translate.sort-order=desc) to find video resources and blog posts, and also refer to [Amazon Translate FAQs](https://aws.amazon.com/translate/faqs/). If you’re new to Amazon Translate, try it out using the [Free Tier](https://aws.amazon.com/translate/pricing/), which offers 2 million characters per month for free for the first 12 months, starting from your first translation request.


## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

