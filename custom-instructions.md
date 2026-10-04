# Custom Instructions for the webMethods Flow skill

This is the primary preferences file for the skill. It lives in the
`extended-skill/` folder next to `SKILL.md`. The skill reads everything under the
**"Your instructions"** heading below and treats it as an override of its
default behavior.

You can also drop additional `.md` files into this `extended-skill/` folder (for
example, a house naming standard or extra examples); the skill loads every
`.md` file in the folder in alphabetical order. See `README.md` for how the
folder works.

What your instructions can and cannot do:
- They CAN change conventions and preferences: naming, default package/folder,
  which post-generation actions to offer, formatting, comments.
- They CANNOT override correctness or safety rules: metadata-driven service
  validation (no invented pub.* services), grammar validity, 0-based FlatFile
  positions, the supported MAP operations (copy / copy-if / set / drop), and
  generating complete, deployable code. If a preference conflicts with one of
  these, the skill follows the correctness rule and tells you why.

---

## Your instructions

### Workspace-first rule — MANDATORY, NO EXCEPTIONS

**All source files (`.flow`, `.java`, or any other artifact) MUST be written to the workspace first using `write_file` or `apply_diff`, then copied to the IS packages directory using `execute_command`.**

- ✅ Write to workspace → copy to IS
- ❌ NEVER write directly to the IS packages directory with `execute_command` heredocs or `cat >`

The workspace is the source of truth. The IS packages directory is the deployment target.



### Field type declarations — CRITICAL rule

**Every new temporary field introduced into the pipeline must have its type
declared.** IS needs the type definition to correctly resolve, copy, and iterate
fields. Missing declarations cause silent failures (null outputs, empty arrays,
wrong types).

The declaration goes in:
- **`mapTarget`** — when the field is being **created/written** (INVOKE output,
  MAP `set` or `copy` target)
- **`mapSource`** — when the field is being **read** (MAP source for `copy` or
  `drop`)

This applies everywhere a new field name appears:

#### INVOKE output — declare every new field in `mapTarget`

```flow
// ✅ CORRECT — every new output field declared in mapTarget
INVOKE pub.string:trim {
    output {
        mapTarget {
            String trimmedInput;
        }
        copy value -> trimmedInput;
    }
}

// ❌ WRONG — IS has no type info for trimmedInput
INVOKE pub.string:trim {
    output {
        copy value -> trimmedInput;
    }
}
```

For arrays, declare the array type:
```flow
INVOKE pub.string:tokenize {
    output {
        mapTarget {
            String[] wordList;    // String[] — not String
        }
        copy valueList -> wordList;
    }
}
```

For Java object types (`java.io.InputStream`, `byte[]`, etc.), declare as `Object`:
```flow
INVOKE pub.client:http {
    output {
        mapTarget {
            Object responseStream;   // java.io.InputStream → Object, never String
        }
        copy body/stream -> responseStream;
    }
}
```

#### Standalone MAP — declare fields in mapSource when reading, mapTarget when writing

```flow
// Reading an existing field to copy it — declare in mapSource
MAP {
    mapSource {
        String concatResult;
    }
    mapTarget {
        String titleCasedWords;
    }
    copy concatResult -> titleCasedWords;
}

// Dropping a field — declare it in mapSource so IS knows the type
MAP {
    mapSource {
        String concatResult;
    }
    drop concatResult;
}

// Dropping an array — declare the array type in mapSource
MAP {
    mapSource {
        String[] wordList;
    }
    drop wordList;
}
```

### Pipeline hygiene — INVOKE leaks

- **Input leak**: webMethods writes every field named in an INVOKE `input { }`
  block (both `copy` targets and `set` values) into the shared pipeline. After
  every INVOKE, add a cleanup MAP that `drop`s all those field names. This
  includes constants like `type`, `delim`, `useRegex`, `beginIndex`, `endIndex`,
  `method`, `loadAs`, `encoding`, `separator`, etc.

- **Output leak**: The source field name of every INVOKE `output { }` copy (e.g.
  `value`, `valueList`, `string`, `document`, `status`) also leaks into the
  pipeline. Drop them in the same cleanup MAP immediately after the INVOKE.

- **Cleanup MAP pattern** — place one after every INVOKE, dropping all leaked
  input field names, all consumed intermediate variables, and the raw service
  output names:
  ```flow
  INVOKE pub.string:trim {
      input {
          copy inputString -> inString;
          set type = "both";
      }
      output {
          mapTarget {
              String trimmedInput;
          }
          copy value -> trimmedInput;
      }
  }
  MAP {
      drop inString;    // leaked INVOKE input target
      drop type;        // leaked set constant
      drop value;       // leaked raw service output name
  };
  ```

- **⚠️ CRITICAL — Drop timing: never drop a field before the last INVOKE that reads it.**
  The cleanup MAP for INVOKE N must only drop fields that are **no longer needed** by
  any subsequent INVOKE. If INVOKE N produces a field that INVOKE N+1 consumes as input,
  that field must **not** be dropped until after INVOKE N+1 completes.

  **Wrong** — drops `requestUrl` after Step 1, but Step 2 needs it:
  ```flow
  INVOKE pub.string:concat { ... }   // Step 1: produces requestUrl
  MAP {
      drop inString1;
      drop inString2;
      drop value;
      drop requestUrl;   // ❌ dropped too early — Step 2 can't read it
  }
  INVOKE pub.client:http {           // Step 2: tries to copy requestUrl → url → gets null
      input { copy requestUrl -> url; }
  }
  ```

  **Correct** — `requestUrl` survives until after the INVOKE that consumes it:
  ```flow
  INVOKE pub.string:concat { ... }   // Step 1: produces requestUrl
  MAP {
      drop inString1;
      drop inString2;
      drop value;
      // requestUrl NOT dropped here — Step 2 still needs it
  }
  INVOKE pub.client:http {           // Step 2: reads requestUrl successfully
      input { copy requestUrl -> url; }
  }
  MAP {
      drop requestUrl;   // ✅ dropped here, after Step 2 has consumed it
      drop url;
      ...
  }
  ```

- **`pub.client:http` specific leaks** — beyond `url`, `method`, `loadAs`, this service
  also writes `encodedURL`, `header`, and `body` into the pipeline unconditionally.
  Always include them in the cleanup MAP:
  ```flow
  MAP {
      mapSource {
          String url;
          String method;
          String loadAs;
          String encodedURL;
          record header;
          record body;
      }
      drop url;
      drop method;
      drop loadAs;
      drop encodedURL;
      drop header;
      drop body;
  }
  ```

### Pipeline hygiene — LOOP output collection

- **LOOP `outputArray` requires a MAP+mapTarget as the final step** in the loop
  body to collect the per-iteration result. Writing to the output variable via an
  INVOKE `output {}` block alone is NOT picked up by the LOOP collector.

  Pattern: write the INVOKE result to an intermediate (declared in `mapTarget`),
  then transfer it to the `outputArray` variable via a MAP with `mapSource` +
  `mapTarget`:
  ```flow
  LOOP {
      inputArray: "/wordList"
      outputArray: "/titleCasedWords"

      INVOKE pub.string:concat {
          output {
              mapTarget {
                  String concatResult;          // declare the new temp field
              }
              copy value -> concatResult;
          }
      }
      MAP { drop value; };

      // Final step: mapSource declares concatResult, mapTarget declares titleCasedWords
      MAP {
          mapSource { String concatResult; }
          mapTarget { String titleCasedWords; }  // LOOP collector picks this up
          copy concatResult -> titleCasedWords;
      }
      MAP {
          mapSource { String concatResult; }
          drop concatResult;
      };
  }
  ```

### Document field resolution

- When copying a nested field out of a document (IData) produced by an INVOKE
  (e.g. after JSON or XML parsing), always declare a `mapSource` record in the
  MAP step to give the parser the structural type it needs to resolve the path:
  ```flow
  MAP {
      mapSource {
          record parsedDoc {
              String fieldYouNeed;
          };
      }
      copy parsedDoc/fieldYouNeed -> outputField;
      drop parsedDoc;
  }
  ```
