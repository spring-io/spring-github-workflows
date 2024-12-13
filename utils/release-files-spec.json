{
  "files": [
    {
      "aql": {
        "items.find": {
          "$and": [
            {
              "@build.name": "${buildname}",
              "@build.number": "${buildnumber}",
              "path": {
                "$match": "org/springframework*"
              }
            },
            {
              "$or": [
                {
                  "name": {
                    "$match": "*.pom"
                  }
                },
                {
                  "name": {
                    "$match": "*.jar"
                  }
                },
                {
                  "name": {
                    "$match": "*.module"
                  }
                },
                {
                  "name": {
                    "$match": "*.asc"
                  }
                }
              ]
            },
            {
              "name": {
                "$nmatch": "*.zip.asc"
              }
            }
          ]
        }
      },
      "target": "nexus/"
    }
  ]
}
