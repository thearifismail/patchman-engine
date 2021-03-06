# Architecture
The project is written as a set of communicating containers. It allows to scale different parts of the application according to needs. It also increases application robustness, because even if one component have some issues (errors, down times), the others are not impacted with that work.

The components are `manager`, `listener`, `evaluator-{upload,recalc}`, `vmaas_sync` and `database`.

### Components
- **manager** - serves information about evaluated systems for a given account via REST API. It allows to display:
  - list of all systems registered for a given account with advisory related stats,
  - list of advisories evaluated for any of these systems,
  - list of all advisories, related to at least one system, with stats (e.g. related systems count).
Manager component depends on database storage only and it's also the only application component directly accesed by application frontend. So it ensures high application stability and availability.

- **listener** - connects to the Kafka service, and listens for messages about newly uploaded archives. When a new archive is uploaded, it updates or creates a record in the `system_platform` database table. Specifically it updates `vmaas_json` column with installed packages list, repos and modules. After that it sends a Kafka message (`patchman.evaluator.upload` topic) to evaluate the system with the `evaluator-upload` component. This component also handles system deleting events (`platform.inventory.events` Kafka topic).

- **evaluator-upload** - connects to the Kafka service (`patchman.evaluator.upload` topic) and listens for evaluation requests from the `listener` component. For each received Kafka message it evaluates system with ID contained in the message. As a evaluation result it updates several database tables (`system_advisories`, `system_platform`, `advisory_account_data`). Evaluation is scaled on two levels, firstly with multiple replicas (more pods) and secondary with multiple goroutines within single pod (set by `CONSUMER_COUNT` environment variable).

- **evaluator-recalc** - same as the `-upload` instance but receives Kafka messages from `vmaas-sync` component (`patchman.evaluator.recalc` Kafka topic) after advisories list is updated in database (`advisory_metadata` database table). After this event all registered systems have to be re-evaluated because the input data for the system evaluation has changed.

- **vmaas-sync** - connects to [VMaaS](https://github.com/RedHatInsights/vmaas), and upon receiving notification about updated data, syncs new advisories into the database, and requests re-evaluation for systems which could be affected by new advisories. It's done via messaging to the `patchman.evaluator.recalc` Kafka topic and receiving by the `evaluator-recalc` component. This component also performs system culling.

- **database** - Stores data about systems, advisories, system advisories and different related data. Detailed description of the component and data layout are in [separate page](database.md).

### Components cooperation schema
![](graphics/schema.png)
