=========================
Popular packages PURLs
=========================

https://github.com/aboutcode-org/popular-package-purls is list of popular open source packages
keyed by Package-URL (PURL).

This is a dataset of popular packages Package-URLs collected from deps.dev using Google's BigQuery
where the number of dependents is used as a measure of popularity.

This is done using this query derived from examples provided at https://docs.deps.dev/bigquery/v1/
that we ran once for each package type: npm, cargo, maven, golang, and pypi ::

    DECLARE
      Type STRING DEFAULT 'NPM';
    
    WITH
      LatestVersions AS (
        SELECT
          System as type,
          Name as name,
          Version as version,
        FROM (
          SELECT
            System as type,
            Name as name,
            Version as version,
            ROW_NUMBER()
              OVER (PARTITION BY
                      System,
                      Name
                    ORDER BY
                      VersionInfo.Ordinal DESC) AS RowNumber
            FROM
              `bigquery-public-data.deps_dev_v1.PackageVersionsLatest`
            WHERE
              System = Type
              AND VersionInfo.IsRelease)
        WHERE RowNumber = 1
        )
    
    SELECT
      D.System as type,
      D.Dependency.Name as name,
      D.Dependency.Version version,
      COUNT(*) AS dependents_count
    FROM
      `bigquery-public-data.deps_dev_v1.DependenciesLatest` AS D
    JOIN
      LatestVersions AS LV
    USING
      (type, name, version)
    WHERE
      D.System = Type
    GROUP BY
      D.System,
      D.Dependency.Name,
      D.Dependency.Version
    ORDER BY
      dependents_count DESC
    LIMIT
      50000;

The output is then post-processed using this simple python script::

    import json
    import os
    ecos = "npm"
    purls = []
    packs = json.load(open(f"json/{ecos}.json"))
    for rec in packs:
         t = rec["type"]
         n = rec["name"]
         v = rec["version"]
         d = rec["dependents_count"]
         purl = f"pkg:{t}/{n}@{v}"
         r = dict(purl=purl, dependents_count=d)
         purls.append(r)
    
    with open("purls.json", "w") as o:
        o.write(json.dumps(purls, indent=2))


Thanks to https://github.com/xeol-io/critical-packages that documented the approach and shared
a repositroy with pre-fetched data.

The data is from deps.dev and is licensed under CC-BY-4.0 as mentioned at https://github.com/google/deps.dev::

    As well as aggregating data, deps.dev generates additional data, including resolved dependencies,
    advisory statistics, associations between entities, etc. This generated data is available under a
    CC-BY 4.0 license.
