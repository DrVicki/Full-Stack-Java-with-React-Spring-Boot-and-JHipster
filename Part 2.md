# Full-Stack-Java-with-React-Spring-Boot-and-JHipster: Part 2 #

**Create Entities to allow CRUD on Photos**

We've talked about how to secure your application, but we haven't done anything with photos! JHipster has a **JDL** (JHipster Domain Language) feature which allows you to model the data in your app and generate entities from it. You can use the **JDL Studio** to do this online and save it locally once you've finished.

My data model for this app has ```Album```, ```Photo```, and ```Tag``` entities and sets up relationships between them. Below is a screenshot of what it looks like in JDL Studio.

![](https://github.com/DrVicki/Full-Stack-Java-with-React-Spring-Boot-and-JHipster/blob/main/hipster-images/03_jdl-studio.png)

Copy the JDL below and save it in a ```flickr2.jdl``` file in the root directory of your project.

```
entity Album {
  title String required
  description TextBlob
  created Instant
}

entity Photo {
  title String required
  description TextBlob
  image ImageBlob required
  height Integer
  width Integer
  taken Instant
  uploaded Instant
}

entity Tag {
  name String required minlength(2)
}

relationship ManyToOne {
  Album{user(login)} to User
  Photo{album(title)} to Album
}

relationship ManyToMany {
  Photo{tag(name)} to Tag{photo}
}

paginate Album with pagination
paginate Photo, Tag with infinite-scroll
```

You can generate entities and CRUD code (Java for Spring Boot; TypeScript and JSX for React) by using the following command:

```
jhipster jdl flickr2.jdl
```

When prompted, type ```a``` to allow overwriting of existing files.

This process will create Liquibase changelog files (to create your database tables), entities, repositories, Spring MVC controllers, and all the **React** code necessary to create, read, update, and delete your entities. It'll even generate JUnit unit tests, Jest unit tests, and Cypress end-to-end tests!

After the process completes, you can restart your app, log in, and browse through the **Entities** menu. Try adding some data to confirm everything works.

By now, you can see JHipster is pretty powerful. It recognized you had an image property of ```ImageBlob``` type and created the logic necessary to upload and store images in your database! **BOOM!**

**Add Image EXIF Processing in Your Spring Boot API**

The ```Photo``` entity has a few properties which can be calculated by reading the uploaded photo's **EXIF** (Exchangeable Image File Format) data. **How do you we that in Java?**

Drew Noakes created a metadata-extractor library to do just that. Add a dependency on Drew's library to your ``pom.xml```:

```
<dependency>
    <groupId>com.drewnoakes</groupId>
    <artifactId>metadata-extractor</artifactId>
    <version>2.16.0</version>
</dependency>
```

Then modify the ```PhotoResource#createPhoto()``` method to set the metadata when an image is uploaded.

```
import com.drew.imaging.ImageMetadataReader;
import com.drew.imaging.ImageProcessingException;
import com.drew.metadata.Metadata;
import com.drew.metadata.MetadataException;
import com.drew.metadata.exif.ExifSubIFDDirectory;
import com.drew.metadata.jpeg.JpegDirectory;

import javax.xml.bind.DatatypeConverter;
import java.io.BufferedInputStream;
import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStream;

import java.time.Instant;
import java.util.Date;

public class PhotoResource {
    ...

    public ResponseEntity<Photo> createPhoto(@Valid @RequestBody Photo photo) throws Exception {
        log.debug("REST request to save Photo : {}", photo);
        if (photo.getId() != null) {
            throw new BadRequestAlertException("A new photo cannot already have an ID", ENTITY_NAME, "idexists");
        }

        try {
            photo = setMetadata(photo);
        } catch (ImageProcessingException ipe) {
            log.error(ipe.getMessage());
        }

        Photo result = photoRepository.save(photo);
        return ResponseEntity
            .created(new URI("/api/photos/" + result.getId()))
            .headers(HeaderUtil.createEntityCreationAlert(applicationName, true, ENTITY_NAME, result.getId().toString()))
            .body(result);
    }

    private Photo setMetadata(Photo photo) throws ImageProcessingException, IOException, MetadataException {
        String str = DatatypeConverter.printBase64Binary(photo.getImage());
        byte[] data2 = DatatypeConverter.parseBase64Binary(str);
        InputStream inputStream = new ByteArrayInputStream(data2);
        BufferedInputStream bis = new BufferedInputStream(inputStream);
        Metadata metadata = ImageMetadataReader.readMetadata(bis);
        ExifSubIFDDirectory directory = metadata.getFirstDirectoryOfType(ExifSubIFDDirectory.class);

        if (directory != null) {
            Date date = directory.getDateDigitized();
            if (date != null) {
                photo.setTaken(date.toInstant());
            }
        }

        if (photo.getTaken() == null) {
            log.debug("Photo EXIF date digitized not available, setting taken on date to now...");
            photo.setTaken(Instant.now());
        }

        photo.setUploaded(Instant.now());

        JpegDirectory jpgDirectory = metadata.getFirstDirectoryOfType(JpegDirectory.class);
        if (jpgDirectory != null) {
            photo.setHeight(jpgDirectory.getImageHeight());
            photo.setWidth(jpgDirectory.getImageWidth());
        }

        return photo;
    }
    ...
}
```
Since you're extracting the information, you can remove the fields from the UI and tests so the user cannot set these values.

In ```src/main/webapp/app/entities/photo/photo-update.tsx```, hide the metadata so users can't edit it. Rather than displaying the ```height```, ```width```, ```taken```, and ```uploaded``` values, hide them. You can do this by searching for ```photo-height```, grabbing the elements (and its following three elements), and adding them to a metadata constant just after ```defaultValues()``` lambda function.

```
const defaultValues = () =>
  ...

const metadata = (
  <div>
    <ValidatedField label={translate('flickr2App.photo.height')} id="photo-height" name="height" data-cy="height" type="text" />
    <ValidatedField label={translate('flickr2App.photo.width')} id="photo-width" name="width" data-cy="width" type="text" />
    <ValidatedField
      label={translate('flickr2App.photo.taken')}
      id="photo-taken"
      name="taken"
      data-cy="taken"
      type="datetime-local"
      placeholder="YYYY-MM-DD HH:mm"
    />
    <ValidatedField
      label={translate('flickr2App.photo.uploaded')}
      id="photo-uploaded"
      name="uploaded"
      data-cy="uploaded"
      type="datetime-local"
      placeholder="YYYY-MM-DD HH:mm"
    />
  </div>
);
const metadataRows = isNew ? '' : metadata;

return ( ... );
```

Then, in the return block, remove the JSX between the image property and album property and replace it with ```{metadataRows}```.

```
<ValidatedBlobField
  label={translate('flickr2App.photo.image')}
  id="photo-image"
  name="image"
  data-cy="image"
  isImage
  accept="image/*"
  validate={{
    required: { value: true, message: translate('entity.validation.required') },
  }}
/>
{metadataRows}
<ValidatedField id="photo-album" name="albumId" data-cy="album" label={translate('flickr2App.photo.album')} type="select">
  <option value="" key="0" />
  {albums
    ? albums.map(otherEntity => (
      <option value={otherEntity.id} key={otherEntity.id}>
        {otherEntity.title}
      </option>
    ))
    : null}
</ValidatedField>
```

In ```src/test/javascript/cypress/integration/entity/photo.spec.ts```, remove the code which sets the data in these fields:

```
cy.get(`[data-cy="height"]`).type('99459').should('have.value', '99459');
cy.get(`[data-cy="width"]`).type('61514').should('have.value', '61514');
cy.get(`[data-cy="taken"]`).type('2021-10-11T16:46').should('have.value', '2021-10-11T16:46');
cy.get(`[data-cy="uploaded"]`).type('2021-10-11T15:23').should('have.value', '2021-10-11T15:23'););
```

Stop your Maven process, run source ```.auth0.env```, then ```./mvnw``` again. Open a new terminal window, set your Auth0 credentials, and run ```npm run e2e``` to make sure everything still works.

```
export CYPRESS_E2E_USERNAME=<auth0-username>
export CYPRESS_E2E_PASSWORD=<auth0-password>
npm run e2e
```

ℹ️ NOTE: If you experience authentication errors in your Cypress tests, it's likely because you've violated **Auth0's Rate Limit Policy**. As a workaround, I recommend you use **Keycloak** for Cypress tests. You can do this by opening a new terminal window and starting your app there using ```./mvnw```. Then, open a second terminal window and run ```npm run e2e```.

If you upload an image you took with your smartphone, the height, width, and taken values should all be populated. If they're not, chances are your image doesn't have the data in it.

Need some sample photos with EXIF data? You can download pictures from EXIF.
[EXIF Samples](https://github.com/ianare/exif-samples)

## Add a React Photo Gallery ##

You've added metadata extraction to your backend, but your photos still display in a list rather than in a grid (like Flickr). To fix that, you can use the [React Photo Gallery](https://github.com/neptunian/react-photo-gallery) component. Install it using npm:

```
npm i react-photo-gallery@8 --force
```

In ```src/main/webapp/app/entities/photo/photo.tsx```, add an ```import``` for Gallery:

```
import Gallery from 'react-photo-gallery';
```
Then add the following just after ```const { match } = props;```. This adds the photos to a set with ```source```, ```height```, and ```width``` information.

```
const photoSet = photoList.map(photo => ({
  src: `data:${photo.imageContentType};base64,${photo.image}`,
  width: photo.height > photo.width ? 3 : photo.height === photo.width ? 1 : 4,
  height: photo.height > photo.width ? 4 : photo.height === photo.width ? 1 : 3
}));
```

Next, add a ```<Gallery>``` component right after the closing ```</h2>```.

```
return (
  <div>
    <h2 id="photo-heading" data-cy="PhotoHeading">
      ...
    </h2>
    <Gallery photos={photoSet} />
    ...
);
```

Save all your changes and restart your app.

```
source .auth0.env
./mvnw
```

Log in and navigate to **Entities > Photo** in the top nav bar. You will see a plethora of photos loaded by [Liquibase](https://www.liquibase.org/) and [faker.js](https://marak.github.io/faker.js/). To make a clean screenshot without this data, I modified ```src/main/resources/config/application-dev.yml``` to remove the "faker" context for Liquibase.

```
liquibase:
  # Append ', faker' to the line below if you want sample data to be loaded automatically
  contexts: dev
  ```
  
Stop your Spring Boot backend and run ```rm -r target/h2db``` to clear out your database (or just delete the ```target/h2db``` directory). Restart your backend.

Now you should be able to upload photos and see the results in a nice grid at the top of the list.
![](https://github.com/DrVicki/Full-Stack-Java-with-React-Spring-Boot-and-JHipster/blob/main/hipster-images/08_photo-gallery.jpeg)

You can also add a "lightbox" feature to the grid so you can click on photos and zoom in. The [React Photo Gallery docs](https://neptunian.github.io/react-photo-gallery/) shows how to do this. I've integrated it into the example for this post, but I won't show the code here for the sake of brevity. You can see the final ```photo.tsx``` with Lightbox added on GitHub or a diff of the necessary changes.

## Make Your Full Stack Java App Into a PWA ##

**Progressive Web Apps**, (PWAs), are the best way for developers to make their webapps load faster and more performant. In a nutshell, PWAs are websites which use recent web standards to allow for installation on a user's computer or device and deliver an app-like experience to those users. To make a web app into a PWA:

  - Your app must be served over HTTPS
  - Your app must register a service worker so it can cache requests and work offline
  - Your app must have a webapp manifest with installation information and icons

For ```HTTPS```, you can set up a certificate for ```localhost``` or (even better), deploy it to production! Cloud providers like **Heroku** will provide you with HTTPS out-of-the-box, but they won't force ```HTTPS```. To force ```HTTPS```, open ```src/main/java/com/auth0/flickr2/config/SecurityConfiguration.java``` and add a rule to force a secure channel when an ```X-Forwarded-Proto``` header is sent.

```
@Override
protected void configure(HttpSecurity http) throws Exception {
    http
        ...
    .and()
        .frameOptions()
        .deny()
    .and()
        .requiresChannel()
        .requestMatchers(r -> r.getHeader("X-Forwarded-Proto") != null)
        .requiresSecure()
    .and()
        .authorizeRequests()
        ...
}
```

The ```workbox-webpack-plugin``` is configured already for generating a service worker, but it only works when running your app with a production profile. This is nice because it means your data isn't cached in the browser when you're developing.

To register a service worker, open ```src/main/webapp/index.html``` and uncomment the following block of code.

```
<script>
  if ('serviceWorker' in navigator) {
    window.addEventListener('load', function () {
      navigator.serviceWorker.register('/service-worker.js').then(function () {
        console.log('Service Worker Registered');
      });
    });
  }
</script>
```

The final feature — a webapp manifest — is included at ```src/main/webapp/manifest.webapp```. It defines an app ```name```, ```colors```, and ```icons```. You might want to adjust these to fit your app.

## Deploy Your React + Spring Boot App to Heroku ##

To deploy your app to Heroku, you'll first need to install the [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli). You can confirm it's installed by running ```heroku --version```.

If you don't have a Heroku account, go to [heroku.com](heroku.com ) and sign up. Don't worry, it's free, and chances are you'll love the experience.

Run ```heroku login``` to log in to your account, then start the deployment process with JHipster:

```
jhipster heroku
```

This will start the [Heroku sub-generator](https://www.jhipster.tech/heroku/) which asks you a couple of questions about your app: what you want to name it and whether you want to deploy it to a US region or EU. Then it'll prompt you to choose between building locally or with Git on Heroku's servers. Choose Git, so you don't have to upload a fat JAR. When prompted to use Okta for OIDC, select ```No```. Then, the deployment process will begin.

You'll be prompted to overwrite ```pom.xml```—type ```a``` to allow overwriting all files.

If you have a stable and fast internet connection, your app should be live on the internet in around six minutes!

```
remote: -----> Compressing...
remote:        Done: 120.9M
remote: -----> Launching...
remote:        Released v7
remote:        https://flickr-2.herokuapp.com/ deployed to Heroku
remote:
remote: Verifying deploy... done.

To https://git.heroku.com/flickr-2.git
 * [new branch]      HEAD -> main

Your app should now be live. To view it, run
   heroku open
And you can view the logs with this command
    heroku logs --tail
After application modification, redeploy it with
    jhipster heroku
Congratulations, JHipster execution, is complete!
Sponsored with ❤️ by @oktadev.
Execution time: 6 min. 19 s.
```

## Configure for Auth0 and Analyze Your PWA Score with Lighthouse ##

To configure your app to work with Auth0 on Heroku, run the following command to set your Auth0 variables on Heroku.

```
heroku config:set \
  SPRING_SECURITY_OAUTH2_CLIENT_PROVIDER_OIDC_ISSUER_URI="https://<your-auth0-domain>/" \
  SPRING_SECURITY_OAUTH2_CLIENT_REGISTRATION_OIDC_CLIENT_ID="<your-client-id>" \
  SPRING_SECURITY_OAUTH2_CLIENT_REGISTRATION_OIDC_CLIENT_SECRET="<your-client-secret>" \
  JHIPSTER_SECURITY_OAUTH2_AUDIENCE="https://<your-auth0-domain>/api/v2/"
  ```
Then, log in to your Auth0 account, navigate to your app, and add your Heroku URLs as valid redirect URIs:

  - Allowed Callback URLs: ```https://flickr-2.herokuapp.com/login/oauth2/code/oidc```
  - Allowed Logout URLs: ```https://flickr-2.herokuapp.com```

After Heroku restarts your app, open it with ```heroku open``` and log in.
