load("@rules_python//python:defs.bzl", "py_binary")
load("@base_pip3//:requirements.bzl", "requirement")
load("//bazel:envoy_build_system.bzl", "envoy_package", "envoy_proto_library")
load("//tools/protodoc:protodoc.bzl", "protodoc_rule")
load("//tools/base:envoy_python.bzl", "envoy_genjson", "envoy_jinja_env", "envoy_py_data")

licenses(["notice"])  # Apache 2

envoy_package()

py_binary(
    name = "generate_empty",
    srcs = ["generate_empty.py"],
    visibility = ["//visibility:public"],
    deps = [":protodoc"],
)

envoy_proto_library(
    name = "manifest_proto",
    srcs = ["manifest.proto"],
)

envoy_py_data(
    name = "protodoc_manifest_untyped",
    src = "//docs:protodoc_manifest.yaml",
)

py_binary(
    name = "manifest_to_json",
    srcs = ["manifest_to_json.py"],
    deps = [
        ":manifest_proto_py_proto",
        ":protodoc_manifest_untyped",
        "//tools/config_validation:validate_fragment",
        "@com_envoyproxy_protoc_gen_validate//validate:validate_py",
        "@com_github_cncf_udpa//udpa/annotations:pkg_py_proto",
        "@com_github_cncf_udpa//xds/annotations/v3:pkg_py_proto",
        "@com_google_protobuf//:protobuf_python",
        requirement("envoy.base.utils"),
    ],
)

genrule(
    name = "manifest",
    outs = ["manifest.json"],
    cmd = """$(location :manifest_to_json) $@""",
    tools = [":manifest_to_json"],
)

envoy_genjson(
    # This prepares the data for `protodoc`
    # As protodoc can be run many times while building the api docs, all
    # data is prepared, validated and normalized before being passed to
    # `protodoc`
    name = "data_srcs",
    srcs = [":manifest.json"],
    # manifest:
    #   validated protodoc manifest (from `docs/protodoc_manifest.yaml`)
    #   contains edge config examples.
    # extensions/contrib_extensions:
    #    extensions metadata (from `source/extensions/extensions_metdata.yaml`
    #    and `contrib/extensions_metadata.yaml`)
    # extension/contrib_extension_categories
    #    index of extensions_categories -> extension (taken from metadata)
    # extension_status_values:
    #    schema for possible extension status (eg wip/stable)
    # extension_security_postures:
    #    schema for possible security postures (eg un/trusted up/downstream)
    filter = """
    {manifest: .[0],
     extensions: .[1],
     contrib_extensions: .[2],
     extension_categories: (
         .[1] | reduce to_entries[] as $item ({};
             .[$item.value.categories[]] += [$item.key])),
     contrib_extension_categories: (
         .[2] | reduce to_entries[] as $item ({};
             .[$item.value.categories[]] += [$item.key])),
     extension_status_values: (
         reduce .[3].status_values[] as $value ({};
             .[$value.name] = $value.description)),
     extension_security_postures: (
         reduce .[3].security_postures[] as $posture ({};
             .[$posture.name] = $posture.description))}
    """,
    yaml_srcs = [
        "//source/extensions:extensions_metadata.yaml",
        "//contrib:extensions_metadata.yaml",
        "//tools/extensions:extensions_schema.yaml",
    ],
)

envoy_py_data(
    name = "data",
    src = ":data_srcs",
)

py_binary(
    name = "protodoc",
    srcs = ["protodoc.py"],
    visibility = ["//visibility:public"],
    deps = [
        ":data",
        ":jinja",
        "//tools/api_proto_plugin",
        "@com_envoyproxy_protoc_gen_validate//validate:validate_py",
        "@com_github_cncf_udpa//udpa/annotations:pkg_py_proto",
        "@com_github_cncf_udpa//xds/annotations/v3:pkg_py_proto",
        "@com_google_protobuf//:protobuf_python",
        requirement("envoy.code.check"),
    ],
)

protodoc_rule(
    name = "api_v3_protodoc",
    deps = [
        "@envoy_api//:v3_protos",
        "@envoy_api//:xds_protos",
    ],
)

py_library(
    name = "rst_filters",
    srcs = ["rst_filters.py"],
)

envoy_jinja_env(
    name = "jinja",
    filters = {
        "rst_anchor": "tools.protodoc.rst_filters.rst_anchor",
        "rst_header": "tools.protodoc.rst_filters.rst_header",
    },
    templates = [
        "templates/empty.rst.tpl",
        "templates/extension.rst.tpl",
        "templates/extension_category.rst.tpl",
    ],
    deps = [
        ":rst_filters",
    ],
)
