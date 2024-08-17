# RFC prompt translation

## Observations

1. Images generated with prompts written in western languages displayed higher level of instruction following compared to those written in eastern languages.

2. Nevertheless, images generated with prompts written in western languages still had trouble following instructions, for example the gender of the model is often disobeyed; i also saw drip pattern inconsistently represented in the outputs, especially worse in french and italian outputs.

3. Images generated with prompts written in eastern languages does not follow the prompt at all.

4. Translated prefix, postfix, and negative prompt for Chinese, Japanese, and Korean have minial effects on the quality of the generation & instruction following.
    - noteably, Korean prompt resulted in nsfw content despite the translated negative prompt includes nsfw as a keyword.

5. For western languages, the difference between using translated vs english fixes is minial. In fact, using the english prefixes may have helped guided the generation, as i no longer observed female models being generated when the prompt asked for male.

6. For eastern languages, using English fixes heavily redirected the generation to be more coherent, although the instructions in the prompt (written in the translated languages) are still not followed.

## Solution
###  Translate Prompts with DeepL api
* **Pros:**
   * **Ease of Implementation:**
      - Leveraging translation APIs is straightforward and doesn't require extensive changes to our infrastructure. We can integrate a service like DeepL translation capabilities or use other translation services.

   * **Speed and Flexibility:**
      - Translation APIs are typically fast and can be adapted quickly to different languages. This allows us to support a wide range of languages without the need for complex model training.

   * **Lower Costs:**
      - Using existing translation services can be cost-effective compared to the significant resources required for training and maintaining a custom model.

   * **Leverages Existing Expertise:**
      - Translation services are built and optimized by experts. its benefits us from their expertise in handling different languages and nuances without needing to build it ourselfs.

   * **Not Policy breaks to translate** 
      - Being a translator and not having `usage policies`, any message can be translated and then validated with our BANNED and NSFW validators.
   *  **Languague Detection**
      - DeepL is quite advanced and can detect the language sent.


* **Cons:**

   * **Quality Variability:**
      - The quality of translations can vary, and the translated prompts might still not fully capture the nuances required for optimal   image generation. This is especially true for complex or idiomatic language constructs ( very unlikely ).
   
   * **Dependency on External Services:**
      - Relying on third-party translation services introduces dependencies that may affect our service’s latency, and could potentially increase costs over time.

# Implementation


* ## Update User Preferences and Job Details Collections

   * ### Add a field to `user_preferences` Collection to store the user's preferred language.

      - **Field Name:** `preferred_language`
      - **Purpose:** To check the user's language preference for sending requests with appropriate headers when the preference is not english. ( FE )

   * ### Add two fields to `job_details` Collection. 

      * Add two fields to the `job_details` struct. In the `generation_parameters`

      * **Field Name:** `is_translated`
         - **Type:** Boolean
         - **Purpose:** To indicate whether the prompt has been translated.

      * **Field Name:** `translated_prompt`
         - **Type:** String
         - **Purpose:** To store the translated to english prompt.


* ## Workflow

  ### 1. New Request Header.  There are two possible approaches for sending language preference information in the request header:

   *  **Frontend Validation with Extra Header:**
      - The frontend checks if the preferred language is not English and sends the request with an additional header, such as:
        ```json
        { "Translation": "true" }
        ```
      - The backend then validates the presence of this header.

   *  **Frontend with Preferred Language Header:**
      - The frontend sends an extra header indicating the preferred language, like:
        ```json
        { "Preferred-Language": "italian" }
        ```
      - The backend validates if the preferred language is not English. This approach is preferable as it provides more explicit language information.

      *Why does the frontend need to send the header?* To avoid making an additional database call in the backend for each generation request.

   ### 2. Process Generation Request in Studio/Retouch

   * #### Header Validation:
      * The backend validates the header. If the preferred language is not English, proceed to make an DeepL API call to translate the text:
      
         ```
        Request:
         POST /v2/translate HTTP/2
         Host: api.deepl.com
         Authorization: DeepL-Auth-Key [AuthKey] 
         User-Agent: YourApp/1.2.3
         Content-Length: 45
         Content-Type: application/json
         {"text":["Ciao mondo!"],"target_lang":"ES"}
        
        Response:
         {
            "translations": [
               {
                  "detected_source_language": "IT",
                  "text": "Hello world!"
               }
            ]
         }
        ```
        ```python
            detected_lang = response['translations'][0]['detected_source_language']
            translated_text = response['translations'][0]['text']
        ```

   * #### Post-Translation Processing:
      - We can validate if `detected_lang != "EN"` so we can update the `coreGenMsg` with the following fields (if the text is english will return the same text): 

        - `CoreGenMsg.IsTranslated = true`
        - `CoreGenMsg.TranslatedPrompt = "my great translated prompt"`

      - The messages will then be sent to Pub/Sub to be received by Gobox.

   ### 3. Message Processing in Gobox

   * #### Predict Request:
      - When sending the prediction request, check the `IsTranslated` field in the `CoreGenMessage`. If it is `true`, use the `TranslatedPrompt` in the request parameters. 
      - The generation will be performed with the translated prompt.

   * #### Save in Job Details:
      - When saving in `job_details`, add the two new parameters to the generation parameters in the struct to be saved.

   ### 4. Operations results
   * Users will see images prompts in the language they use to generate the images since we dont changing nothing. Only adding.
   * Will have two new generation_paramaters fields in job_details
   * We have the languague preferences of the user in the user_preferences
---
```json
Note: Given the current backend design, these all changes can be implemented with no complexity at all.
Note: Frontend changes will be minimal.
Note: Backend changes will be minimal.
```
---
## Achievements:
   * Users can submit prompts in their preferred language.
   * Provides more information about user preferences.
## To-Do's After This Change:
   * Implement a language preference question in the frontend to new users.
   * Survey active users about their language preferences.
