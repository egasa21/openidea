# Spring configuration navigation

`openidea.java.configuration` bridges YAML / properties to the existing JDT LS
Java model. It is bundled with the Java usages extension; it does not install a
second language server, run a build, change the application's dependencies, or
connect to external configuration services.

## Editor behavior

- **Go to Definition** on a YAML / properties key opens binding members and
  literal consumers in Java. Multiple matches use the existing location chooser.
- **Go to Definition** inside a Java `@Value` placeholder or a constant
  `Environment.getProperty` / `getRequiredProperty` argument opens matching
  configuration keys. Ordinary Java definition navigation remains native.
- **Find Usages** joins configuration declarations, property consumers, and Java
  references to associated binding members. Java's existing reference results and
  read/write classifications are retained.
- Open editor contents take precedence over disk. All matching profiles/documents
  are shown; navigation does not claim to know the runtime effective value.
- Missing or older extension bundles preserve native language-server behavior.
  YAML aliases still use the YAML server's anchor navigation.

## Association sources

The Java command resolves annotation and method identities with JDT bindings:

- `@ConfigurationProperties` on classes and factory methods, including inherited
  members, nested objects, records, constructor-only parameters / `@Name`, arrays,
  collections and maps. Binder keys use relaxed spelling while retaining path
  boundaries. Indexed collection positions and arbitrary map keys use wildcards.
- `@Value("${key:default}")`, including constant annotation values and nested
  default placeholders.
- Literal or compile-time constant keys passed to Spring's `PropertyResolver`
  methods, including through `Environment`. Unrelated methods with the same name
  and non-Spring annotations are excluded.
- Dependency JAR `META-INF/spring-configuration-metadata.json` and
  `additional-spring-configuration-metadata.json`. The closest group, source
  type, and source method locate a member or its owning class. Source attachments
  and JDT class-file/decompiler navigation use the existing virtual-document path.
- The same metadata files in source roots and `src/main/resources`.

YAML key ranges come from the bundled tree-sitter grammar, including nested/flow
mappings, sequences, quoted keys and multiple documents. Properties support
escaped keys, Unicode escapes and logical-line continuations. Ranges crossing
IPC are UTF-16. No new IPC type or frontend provider is required.

## Scope and costs

The command uses the selected JDT project's resolved package roots, including
source dependencies and its dependency JARs. Configuration search includes those
modules, skips nested independent modules and build output, and honors ignore
files for disk discovery. It does not guess a global association from matching
text in unrelated projects.

Indexing is lazy. YAML/properties navigation and searches for verified configuration
associations build a compact snapshot;
ordinary typing, code-lens probes and bulk scoped reference analysis do not start indexing. At most eight project snapshots are
retained, with a shared 100,000-association retention cap. Records contain property names, element handles and source locations;
ASTs and bindings are discarded after indexing. A lexical prefilter avoids AST
construction for ordinary Java units, but every accepted association is resolved
semantically. Warm requests reuse the snapshot. Java model/reconcile changes
invalidate it conservatively, including dependent-type changes; open working-copy
content fingerprints also prevent stale answers before reconciliation. This is
**snapshot invalidation**, not a fully incremental Java semantic index.

Java selections are checked before building that snapshot. Definition probes first
resolve the selected symbol; ordinary methods and non-constant variables return directly
to native JDT navigation. Possible constant expressions get a syntax check without bindings.
A selected `@Value` expression or Spring property-lookup argument resolves
only its own compilation unit and returns keys for the existing configuration-file search.
It does not build the project-wide configuration index.

Find Usages resolves the selected Java element first. Locals and parameters skip the
configuration index. Potential binding members are checked against JDT's indexed
`@ConfigurationProperties` declarations, including nested/inherited members and factory
return types. Metadata checks resolve only matching qualified type names, property names or factory methods,
rather than every metadata entry. Only a verified association starts the full index
and merges configuration usages. At the source-candidate limit, the existing bounded
index remains the conservative fallback. These eligibility checks still have a cost, especially
for fields/accessors in projects with many binding declarations; cold JDT initialization
and ordinary Java reference searches also remain owned by JDT LS.

Metadata/JAR timestamps are checked before reuse. The client caches parsed disk
configuration files by modification time and size (at most 100,000 cached keys), overlays open buffers, and
performs bounded discovery/parsing on a blocking worker. It does not continuously
watch or parse an additional copy of the entire workspace.

Bounds: 20,000 Java units, 100,000 associations per snapshot, 2 MiB per candidate
metadata/configuration file (Java source: 2M UTF-16 units), 1,024-character property names, 2,000 configuration files/results, 50,000 filesystem
entries per root, and depth limits on recursive bindings/YAML. Java requests and
references are cancellable. File/association scan limits and unreadable files carry the
existing incomplete-result indicator; unsupported syntax keys are omitted. First navigation can take seconds on a
cold project; classpath import and source download remain owned by JDT LS.

## Deliberate limits

This is Java/Spring property navigation, not full IntelliJ Spring support. It
cannot infer dynamically constructed names, custom configuration APIs/annotations,
remote-only configuration, arbitrary SpEL evaluation, YAML merge/alias expansion,
or custom Spring converters that change the binding shape. Kotlin is not indexed
by this Java bundle. YAML-only quoted-string escape forms that cannot be decoded
safely are skipped. Properties surrogate-pair escapes are not decoded as keys.
A missing source attachment may navigate to the decompiled owning class when JDT
cannot provide an exact member range. Binding members are navigation candidates;
the feature does not validate whether Spring will successfully bind a bean at
runtime or resolve profile/precedence rules.

## Verification

Use the JDT LS distribution / Java version documented in [README](README.md).

```sh
python3 plugins/java-usages/build.py /path/to/jdtls
python3 plugins/java-usages/tests/configuration.py /path/to/jdtls
OPENIDEA_CONFIGURATION_STRESS=1 python3 plugins/java-usages/tests/configuration.py /path/to/jdtls
OPENIDEA_SPRING_BOOT_JAR=/path/to/spring-boot-autoconfigure.jar python3 plugins/java-usages/tests/configuration.py /path/to/jdtls
cargo test -p ide-language -p ide-lsp
```

The isolated real-server harness covers custom bindings, inheritance, nesting,
collections/maps, records, constructor aliases, factory methods, consumers,
constant keys, negative identity matches, dependency metadata/source locations,
reverse queries, and unsaved-change/close invalidation. Optional stress fixtures
add 3,000 ordinary units and 100 binding classes; output includes cold/warm/edit
latencies for the JDT command only, excluding server import, UI and configuration-file discovery. The optional actual Spring Boot JAR checks `server.port`,
`spring.datasource.url` and `spring.jackson.date-format` through dependency
metadata. No user's project is compiled or modified by the harness.
Ordinary Java type, method, field, accessor, local and parameter probes run before the
configuration index is built and again after an edit. They must return no configuration
keys, report zero indexed units, and retain native definitions/references. The harness
also verifies binding declarations in another source unit and metadata-only source/library
members; `ordinaryJavaMs` records the routing-query timings separately from native lookups.

Rust tests cover configuration syntax/ranges and the actual manager bridge:
YAML navigation without YAML definition capabilities, Java native fallback,
reference merging, profiles, open-buffer overlays and sibling-module exclusion.
Existing Java usages / read-write integration harnesses remain regression checks.
