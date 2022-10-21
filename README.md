## Enterprise Translation - A foundational use case for customizing Amazon Translate

This post provides an example of how to build a centralized customizable translation solution using Amazon Translate, Amazon API Gateway, AWS Lambda and Amazon DynamoDB. For details about the use-case, please refer to the [blog post](https://aws.amazon.com/blogs/machine-learning/a-foundational-use-case-for-customizing-amazon-translate/).

The solution uses the native features of Amazon Translate, including real-time translation, automatic source language detection (powered by Amazon Comprehend), and custom translation using custom terminology. You can expose these features as one simple /translate API using API Gateway. In order for custom terminology to work, you also need to upload the terminology files to Amazon Translate. Therefore, the API /customterm is exposed to do that.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

