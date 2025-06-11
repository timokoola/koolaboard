# Terminal Board and extremely simple CMS

## Use case description

As a member of a family I want a quick access to daily information by the outside door of our flat. The information is available on a TRMNL board by the door and it is easily glanceable while heading out. The part of information is user updateable on the go with mobile browser. Browser UI is fast to use on a mobile browser. I value speed, simplicity and glanceability.

## System description

This is a simple node.js based web app / API that runs on Google Cloud. The system provides following functionality:

- simple web ui with four panels layed out in 2 x 2 grid (requires GCP cloud identity authentication to access)
- web ui to update two of the panels with user provided information (again, requires authentication)
- same information for TRMNL (no authentication just an url) to fetch the needed data for the device as part of the API (JSON). API serves the JSON item from the database
- API for updating the user generated panels
- API for triggering updates
- cloud run function that updates the data for the UI in the database and cleans up the database of obsolete entries
- both the UI and the API have /version endpoint that shows the current build information stored in a separate build step. Version endpoints are unauthenticated

Panels have following functionality:

- Panel "notes": Simple notes or reminders that are a line of text provided by the user. There is a limited, configurable number of items and users can freely replace the text content. CMS UI provides a method to edit these items. Title and number of items are configurable by environment variables. Any saved edit triggers the update of all data. Admin user can lock individual items so that regular users can't update them
- Panel "calendar": similar to notes but items have a start date (only date, no time), and an optional end date (when no end date is set, assume the start and end date are the same). Calendar items are not shown after 23:59 on the end date. Number of items are configurable by environment variables. Title word is also configurable. Title contains also the current date in a Finnish locale with no year and the current weather information (Material UI icon for weather now) and the current temperature in Celsius. Any saved edit triggers the update of all data
- Panel "stop 1": Stop information for a HSL (Helsinki area) traffic stop. The panel shows next departures from the stop. First item is bigger than next two and those are bigger than rest of the items on the list. If departure is cancelled for any reason a warning icon is shown and the text is stricken through. The title is the stop short code and the name of the stop in Finnish. HSL Reittiopas API is used and the API key is stored as an environment variables. The stop id is a configurable item
- Panel "stop 2": identical to stop 1 panel

Order of panels is configurable in the environment variables, the default order is calendar in top left corner, stop 1 top right, stop 2 bottom left and notes bottom right.

## Integrations

- HSL Digitransit API as documented in [APIs | Digitransit](https://digitransit.fi/en/developers/apis/) and specifically [Stops | Digitransit](https://digitransit.fi/en/developers/apis/1-routing-api/stops/). Primary Key and Secondary keys are stored as environment variables.
- Open Weather Map as documented in [OpenWeatherMap API guide - OpenWeatherMap](https://openweathermap.org/guide). The API key is stored as an environment variable.
- TRMNL APIs as documented here [Overview | TRMNL API](https://docs.usetrmnl.com/go). Specifically the UI parts: [Screens](https://docs.usetrmnl.com/go/private-plugins/templates) and [Plugins](https://docs.usetrmnl.com/go/private-plugins/create-a-screen). Again, all the keys are to be stored as environment variables.

## Configurations

Direnv [direnv – unclutter your .profile | direnv](https://direnv.net/) is used to store the local configuration. The .envrc and additional rc files defined below are gitignored. The envrc if the single source of truth for all configuration. The cloud configurations are updated from the local configuration. Settings in .envrc also ensure that correct programming environments are in use. Any script that is used to manage system always validates that all needed environment variables are set and that gcp auth login is done and the gcp project is properly set to a correct project. All scripts are idempotent: if the resource was already created the creation is not to be retried.

/scripts/create_skeleton_envrc creates .envrc in the main level if the file doesn't exist. All the environment variables are set to dummy values

### Cloud configuration

All the GCP related environment variables are defined in .cloud.envrc that is sourced by the main .envrc file and is also gitignored.

### GCP project

GCP Project is provisioned by shell scripts in the /infra directory. The script /infra/create_project.sh creates the GCP project and sets the GCP_PROJECT environment variable in .cloud.envrc. There is one GCP project for all resources. This design decision follows from relative simplicity of the system. There are no separate testing environments.

### Domain and DNS

Domain to be used is in the environment variable SERVICE_DOMAIN. GCP Cloud DNS is used to manage it.

### User management and configuration

Users and access to editing UI is managed by GCP Cloud Identity. Users are defined in file users.json that is gitignored contains the users that are allowed to access the system. JSON structure is as follows:

```json
[
  {
    "user": "timo@example.com",
    "identity": "apple",
    "admin": true
  },
  {
    "user": "timo.toinen@example.com",
    "identity": "google",
    "admin": false
  }
]
```

Script /helpers/create_user_skeleton.sh generates the skeleton file for users.

Script /infra/create_users.sh creates users in cloud identity. Expected number of users in the system is very low (10-20 identities and 5-6 people).

### GCP Cloud resources

Following resources are used:

- Cloud run (hosting the node.js web app and the API in a single container)
- Cloud run functions: the updater function
- Firestore: the database for storing the state
- Cloud Build / Artifact Registry for building container images for cloud run and updater

The script /infra/create_infra.sh sets up the environment.

### Builds

Builds are managed GCP Cloud build. Needed cloud build configuration files and scripts to trigger builds are in directory /builder.

As part of the build the current version is stored as static asset and included in the containers. This information contains the current commit hash and the commit message along with a timestamp. This information is presented in the /version paths of the UI and the API.

## Application Design and Implementation Considerations

- Node 22 is used in all parts of the Application
- Web application is built on vanilla CSS and HTML. All javascript on page is also vanilla.
- Speed is the main design goal of the UIs
- Admin UI needs to be usable in a mobile browser. User logs in with either Apple ID or Google identity
- Updates happen every 3 minutes from 5:30 am to midnight Finnish time, whenever user saves anything and every 30 minutes between midnight and 5:30am. Weather information is fetched once an hour. Updates are handled by a cloud run function triggered on schedule and by the editor UI saves.
- HSL digitransit graphql-queries and Weather api calls are accessible for both the service code and the test scripts
- Data models are in a separate code module to be shared among the node.js backend, cloud run function and test scripts
- README.md contains basic and succint information on how to set up the system along with a brief information about the main use case of the system

## Integration test scripts

There are set of test scripts in /tests directory that can be used to determine the state of the system during development and troubleshooting

/tests/basics.sh validates that environment variables are set and that cloud project exists and that the gcp auth and project are correct.
/tests/dns.sh prints out the domain information
/tests/identity.sh validates GCP cloud identity is correct and pretty prints the users in the system
/tests/hsl.sh validates the HSL information for making the needed transport info queries and pretty prints the next departures on stops 1 and 2
/tests/weather.sh validates the weather API information and pretty prints the current weather in Helsinki, Finland
/tests/trmnl.sh calls the API running on GCP cloud run like the TRMNL service does to validate that system works as intended
/tests/system.sh makes a call to web ui /version and API /version to validate that the system is running

## Implementation order and validation

Commit after each step when the change has been validated by a human. Use conventional commit format

1. The envrc helper shell
2. Project creation script
3. Infra scripts
4. Integration to HSL and Weather service and associated test scripts
5. Web application and service code along with test scripts
6. TRMNL integration API and test script
