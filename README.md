# renovate-depName-mismatch-repro
Attempt to mininally reproduce a problem with regex manager depName mismatch errors.

## Background:
Our GitLab CI configuration file contains a number of variables that specify container image URLs, so that those images can be referenced via the variables in the "image:" sections of various jobs that use the same image without duplication.  The standard managers do not interpret those variables, so we are attempting to use a regexManager to update these images.

**Given** a .gitlab-ci.yml file with the following variable declaration:
```yaml
variables:
  CUSTOM_IMAGE_RHEL7_X86_64: registry.gitlab.mycompany.com:443/our_group/a_subgroup/our_project/some_folder/image_name_ending_with_rhel7-x86_64:6
```

**And given** the following in our renovate.json:
```json
{
    "regexManagers": [
        {
            "description": [
                "All GitLab CI variables starting with 'CUSTOM_IMAGE_' referring to ",
                "images served from the our_project project container registry"
            ],
            "fileMatch": ["^\\.gitlab-ci\\.yml$"],
            "matchStrings": [
                "\\n\\s*CUSTOM_IMAGE_\\S+: (?<registryUrl>registry.gitlab.mycompany.com:443)/(?<depName>our_group/a_subgroup/our_project/some_folder/[^:]*?):(?<currentValue>\\d+)\\s"
            ],
            "datasourceTemplate": "docker"
        }
    ]
}
```

## Expected behavior:

We expect that the tag (":6") in the variable value should be changed to the latest available tag (i.e., ":9").


## Current behavior:

The update fails, with the following message in the debug log:
```
"manager":"regex","packageFile":".gitlab-ci.yml","currentDepName":"our_group/a_subgroup/our_project/some_folder/image_name_ending_with_rhel7-x86_64","newDepName":"our_group/a_subgroup/our_project/some_folder/image_name_ending_with_rhel7-x89_94","msg":"depName mismatch"
```

## Discussion:
So apparently some step in the process is applying the change from "6" to "9" to the depName instead of (or in addition to) applying it to the tag.

This behavior seems similar if not identical to the one documented in [Issue 8061](https://github.com/renovatebot/renovate/issues/8061), but that issue is marked as fixed.  I don't know if there could have been a regression that caused the issue to reappear, or if this is a different root cause with the same symptom.  In some discussions I found regarding that issue I saw a suggestion to add an autoReplaceStringTemplate setting to the regexManager, so I tried adding this line:
```json
   "autoReplaceStringTemplate": "{{{depName}}}:{{{newValue}}}",
```

This changed the error somewhat but didn't solve it.  The resulting "depName mismatch" error indicated that with this addition to the configuration, the depName was being changed to a different depName in the same file.
```
"manager":"regex","packageFile":".gitlab-ci.yml","currentDepName":"our_group/a_subgroup/our_project/some_folder/image_name_ending_with_rhel7-x86_64","newDepName":"our_group/a_subgroup/our_project/some_folder/image_name_ending_with_ubuntu_20_04-x86_64","msg":"depName mismatch"
```

In this replication repo I am demonstrating both the original problem and the unsuccessful workaround at the same time, using variables starting with "IMAGE_TEST_" and "WORKAROUND_TEST_" respectively.
