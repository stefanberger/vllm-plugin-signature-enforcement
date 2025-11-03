# Using vLLM with AI Model Signature Enforcement

vLLM supports a plugin architecture for validation of AI models and LoRA
adapters. This repository hosts a plugin that enables this validation using
signature enforcement in order to ensure integrity and provenance of AI
models. A policy that the user passes to the plugin provides the flexibility
for which AI models' signatures are verified and how the verification is
done.

The signature enforcement of this plugin builds on signatures created by the
[model signing library](https://github.com/sigstore/model-transparency).

## Installing the Plugin

Note: Installation using `pip` is not possible, yet.

To install the plugin it is necessary to first activate the `venv` of vLLM
and then install the plugin into its `venv`:

```bash
cd venv
. venv/bin/activate
cd ../vllm-plugin-signature-enforcement
pip install -e .
```

### Security Policy for Signature Verification

The signature verification part of the security policy allows to verify
signatures created by the
[model signing library](https://github.com/sigstore/model-transparency).
All of the signature verification methods of this library are support:

| Verification Method | Required Parameters  |
|---------------------|----------------------|
| sigstore    | identity, identity_provider |
| certificate | certificate_chain |
| key         | public_key |

The parameters column in the above table shows the parameters needed
for each one of the verification methods.

### Signature Verification on AI Models

To enforcement signature verification on AI models it is necessary to
write a signature enforcement policy. This policy allows to specify which
AI models have their signature verified and which signature verification
method is to be used. A signature enforcement policy that expects the
'sigstore' type of signature on an AI model may look like this:

```text
{
  "policy": {
    "signatures": {
      "signers": {
        "granite-verify": {
          "verification_method": "sigstore",
          "identity": "Granite-verify@ibm.com",
          "identity_provider": "https://sigstore.verify.ibm.com/oauth2"
        }
      },
      "models": {
        "/tmp/test/granite-4.0-micro/": {
          "signer": "granite-verify"
        }
      }
    }
  }
}
```

The above policy shows a single AI model with the local file path
`/tmp/test/granite-4.0-micro` under its `models` object. It references the
signer `granite-preview`, which is mentioned above under the `signers` object.
It requires the sigstore verification method and the identity that is expected
to have signed the model must be `Granite-verify@ibm.com` using the
identity_provider `https://sigstore.verify.ibm.com/oauth2`.
Since no other models are enumerated in the `models` object, it will not be
possible to load any other models with this policy. The vLLM log will show
whether signature verification succeeded or failed.

Note that if the `"models"` object is missing in a policy that all models
can be loaded since no signature verification will be done.

Extending the above shown policy to one that also covers the other signing
methods leads to a more complex policy:

```text
{
  "policy": {
    "signatures": {
      "signers": {
        "granite-verify": {
          "verification_method": "sigstore",
          "identity": "Granite-verify@ibm.com",
          "identity_provider": "https://sigstore.verify.ibm.com/oauth2"
        }
        "cert-signer": {
          "verification_method": "certificate",
          "certificate_chain": ["/tmp/baz/cert1.pem", "/tmp/baz/cert2.pem"]
        },
        "key-signer": {
          "verification_method": "key",
          "public_key": "/tmp/baz/pubkey.pem"]
        },
        "no-signer": {
           "verification_method": "skip"
        }
      },
      "models": {
        "/tmp/test/granite-4.0-micro/": {
          "signer": "granite-verify"
        }
        "regex:/tmp/(baz1|baz2)(/)?": {
          "signer": "cert-signer",
          "log_fingerprints": true,
          "ignore_paths": ["foo", "bar"]
        },
        "/tmp/test/": {
          "signer": "no-signer"
        },
        "regex:.*": {
          "signer": "key-signer"
        }
      }
    }
  }
}
```

The above policy introduces 3 more signers with the methods 'certificate',
'key', and 'skip' along with their required parameters. The 'skip' method
allows access to unsigned AI models or to simply skip signature verification.
The 'certificate' and 'key' methods require each a different set of
parameters through which they reference PEM-formatted certificates or a
public key respectively. The names of the parameters are shown in the table
above.

The 'models' object holds 3 more paths, of which two are regular
expressions. The first regular expression `/tmp/(baz1|baz2)(/)?` covers
a signature verification rule for the following paths:

- `/tmp/baz1`
- `/tmp/baz1/`
- `/tmp/baz2`
- `/tmp/baz2/`

The additional log_fingerprints parameter indicates whether to log
the fingerprints of the certificates when verifying the signature.

The model path `/tmp/test/` references the `no-signer` signer, and therefore
will skip signature verification on the model found under `/tmp/test/`.

The last model path is again a regular expression `.*` that covers all
(remaining) paths and since it references `key-signer`, it will require that
all these models will have to pass signature verification with the
referenced public key.

To select the signature verification parameters for a particular model,
vLLM will first try to perform an exact path match of the model path from
the command line with the policy. Note that in this case the paths `/foo/bar`
and `/foo/bar/` will both match the path `/foo/bar/` in the policy. All
paths that are not regular expression should therefore end with a '/' in the
policy. If no matching path could be found in the policy, then vLLM will try
to match the regular expressions against the model path from the vLLM command
line. The first matching regular expression will be used to select the
signature verification parameters.

The following table shows additional key-value pairs that can be used on
signer objects:

| Key               | Value Type   | Optional/Mandatory| Purpose |
|------------------|---------------|-----------------|---------|
| signer           | name               | Mandatory | Use this field to reference a signer |
| signature        | filename           | Optional  | The name of the signature file; default is `model.sig` |
| log_fingerprints | true or false      | Optional  | Log fingerprints of certificates used for signature verification; only useful if 'certificate' method is used |
| ignore_git_paths | true or false      | Optional  | Ignore git related files such as `.git`, `.gitignore`, and `.gitattributes` in the model path |
| use_staging      | true or false      | Optional  | Sigstore staging servers were used for signing with the 'sigstore' method |
| ignore_path      | list of file paths | Optional  | Files to ignore when verifying the signature, e.g. `['foo','bar']` |
| ignore_unsigned_files | true or false | Optional  | Ignore files not covered by the signature; default is 'false' |

All files mentioned in this table are assumed relative to the model path
unless they are given as absolute paths (starting with '/').


### Signature Verification on LoRA Adapters

The rules for selecting the signature verification parameter of a LoRA
adapter are the same as those for AI models. The difference is that LoRA
adapters are enumerated in their own 'loras' object as shown in the
following policy:

```text
{
  "policy": {
    "signatures": {
      "signers": {
        "signer1": {
          "verification_method": "sigstore",
          "identity": "foo@bar.com",
          "identity_provider": "http://baz.com/oauth2"
        }
      },
      "loras": {
        "/tmp/bar/" {
          "signer": "signer1"
        }
      }
    }
  }
}
```

This policy enforces 'sigstore' signature verification on the LoRA adapter
in the `/tmp/bar/` directory. Since no other paths are given, it will not
be possible to load any other LoRA adapters. Further, since no `"models"`
object is provided, no signature verification will be performed on regular
AI models.


## VLLM Command Line for Signature Enforcement

To start vLLM with signature enforcement support add the name of the plugin to
the comma-separated list of plugins in the `VLLM_PLUGINS` environment variable
along with the plugin-specific environment variable to pass the signature
enforcement policy:

```bash
VLLM_PLUGIN_SIGNATURE_ENFORCEMENT_POLICY=mypolicy.json \
VLLM_PLUGINS=lora_filesystem_resolver,signature_enforcement \
    vllm serve SOME_MODEL
```


### Signature Verification Quick-start Example

The following example clones an AI model and enforces that it was signed by
Granite-validate@ibm.com.

```bash
cd /tmp
mkdir test
cd test
git clone https://huggingface.co/ibm-granite/granite-4.0-micro
cat <<_EOF_ >policy.json
{
  "policy": {
    "signatures": {
      "signers": {
        "granite-verify": {
          "verification_method": "sigstore",
          "identity": "Granite-verify@ibm.com",
          "identity_provider": "https://sigstore.verify.ibm.com/oauth2"
        }
      },
      "models": {
        "/tmp/test/granite-4.0-micro/": {
          "signer": "granite-verify"
        }
      }
    }
  }
}
_EOF_

VLLM_PLUGINS=signature_enforcement \
VLLM_PLUGIN_SIGNATURE_ENFORCEMENT_POLICY=/tmp/test/policy.json \
vllm serve /tmp/test/granite-4.0-micro
```

vLLM shows the following line in its log:

```text
(EngineCore_DP0 pid=3095) INFO 10-06 19:46:14 [plugins.py:70] Successfully validated /tmp/test/granite-4.0-micro
```
