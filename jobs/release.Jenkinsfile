def pipeutils, pipecfg, official
node {
    checkout scm
    pipeutils = load("utils.groovy")
    pipecfg = pipeutils.load_pipecfg()
    def jenkinscfg = pipeutils.load_jenkins_config()
    official = pipeutils.isOfficial()
}

properties([
    pipelineTriggers([]),
    parameters([
      choice(name: 'STREAM',
             choices: pipeutils.get_streams_choices(pipecfg),
             description: 'Fedora CoreOS stream to release'),
      string(name: 'VERSION',
             description: 'Fedora CoreOS version to release',
             defaultValue: '',
             trim: true),
      string(name: 'ARCHES',
             description: 'Space-separated list of target architectures',
             defaultValue: "x86_64" + " " + pipecfg.additional_arches.join(" "),
             trim: true),
      booleanParam(name: 'ALLOW_MISSING_ARCHES',
                   defaultValue: false,
                   description: 'Allow release to continue even with missing architectures'),
      // Default to true for AWS_REPLICATION because the only case
      // where we are running the job by hand is when we're doing a
      // production release and we want to replicate there. Defaulting
      // to true means there is less opportunity for human error.
      booleanParam(name: 'AWS_REPLICATION',
                   defaultValue: true,
                   description: 'Force AWS AMI replication'),
      string(name: 'COREOS_ASSEMBLER_IMAGE',
             description: 'Override coreos-assembler image to use',
             defaultValue: "coreos-assembler:main",
             trim: true)
    ]),
    durabilityHint('PERFORMANCE_OPTIMIZED')
])

// no way to make a parameter required directly so manually check
// https://issues.jenkins-ci.org/browse/JENKINS-3509
if (params.VERSION == "") {
    throw new Exception("Missing VERSION parameter!")
}

currentBuild.description = "[${params.STREAM}][${params.ARCHES}] - ${params.VERSION}"

// Get the list of requested architectures to release
def basearches = params.ARCHES.split() as Set

def stream_info = pipecfg.streams[params.STREAM]

// We just lock here out of an abundance of caution in case somehow two release
// jobs run for the same stream, but that really shouldn't happen. Anyway, if it
// *does*, this makes sure they're run serially.
// Also lock version-arch-specific locks to make sure these builds are finished.
def locks = basearches.collect{[resource: "release-${params.VERSION}-${it}"]}
lock(resource: "release-${params.STREAM}", extra: locks) {
    cosaPod(cpu: "1", memory: "2Gi",
            image: params.COREOS_ASSEMBLER_IMAGE) {
    try {

        def gcp_image = ""
        def ostree_prod_refs = [:]

        // Fetch metadata files for the build we are interested in
        stage('Fetch Metadata') {
            withCredentials([file(variable: 'AWS_CONFIG_FILE',
                                  credentialsId: 'aws-build-upload-config')]) {
                def ref = pipeutils.get_source_config_ref_for_stream(pipecfg, params.STREAM)
                shwrap("""
                cosa init --branch ${ref} ${pipecfg.source_config.url}
                cosa buildfetch --artifact=ostree --build=${params.VERSION} \
                    --arch=all --url https://s3.amazonaws.com/rhcos-jlebon/buildupload/builds
                """)
            }
        }

        def builtarches = shwrapCapture("jq -r '.builds | map(select(.id == \"${params.VERSION}\"))[].arches[]' builds/builds.json").split() as Set
        assert builtarches.contains("x86_64"): "The x86_64 architecture was not in builtarches."
        if (!builtarches.containsAll(basearches)) {
            if (params.ALLOW_MISSING_ARCHES) {
                echo "Some requested architectures did not successfully build! Continuing."
                basearches = builtarches.intersect(basearches)
            } else {
                echo "ERROR: Some requested architectures did not successfully build"
                echo "ERROR: Detected built architectures: $builtarches"
                echo "ERROR: Requested base architectures: $basearches"
                currentBuild.result = 'FAILURE'
                return
            }
        }

        def registry_repos = pipeutils.get_registry_repos(pipecfg, params.STREAM)

        // [config.yaml name -> [meta.json artifact name, meta.json toplevel name, tag suffix]]
        // The config.yaml name is the name used in the `registry_repos` object. The
        // meta.json artifact name is the "cosa name" for the artifact (in the `images`
        // object). The top-level name is the key inserted with the pushed info. The tag
        // suffix is needed to avoid the containers clashing in the OCP ART monorepo. It
        // could be made configurable in the future. For now since FCOS doesn't need it and
        // OCP ART doesn't actually care what the tag name is (it's just to stop GC), we
        // hardcode it.
        def push_containers = ['oscontainer': ['ostree', 'base-oscontainer', ''],
                               'extensions': ['extensions-container', 'extensions-container', '-extensions'],
                               'legacy_oscontainer': ['legacy-oscontainer', 'oscontainer', '-legacy']]

        // filter out those not defined in the config
        push_containers.keySet().retainAll(registry_repos.keySet())

        // filter out those not built. this step makes the assumption that if an
        // image isn't built for x86_64, then we don't upload it at all
        def meta_x86_64 = readJSON file: "builds/${params.VERSION}/x86_64/meta.json"
        def artifacts = meta_x86_64.images.keySet()

        // in newer Groovy, retainAll can take a closure, which would be nicer here
        for (key in (push_containers.keySet() as List)) {
          if (!(push_containers[key][0] in artifacts)) {
            push_containers.remove(key)
          }
        }

        if (push_containers) {
          stage("Push Containers") {
            parallel push_containers.collectEntries{configname, val -> [configname, {
              withCredentials([file(variable: 'REGISTRY_SECRET',
                                    credentialsId: 'oscontainer-push-registry-secret')]) {
                  def repo = registry_repos[configname]
                  def (artifact, metajsonname, tag_suffix) = val
                  def arch_args = basearches.collect{"--arch ${it}"}.join(" ")
                  def tag_args = " --tag=${params.STREAM}${tag_suffix}"
                  if (registry_repos.add_build_tag) {
                      tag_args += " --tag ${params.VERSION}${tag_suffix}"
                  }
                  shwrap("""
                  cosa push-container-manifest --auth=\${REGISTRY_SECRET} \
                      --repo=${repo} ${tag_args} \
                      --artifact=${artifact} --metajsonname=${metajsonname} \
                      --build=${params.VERSION} ${arch_args}
                  """)

                  def old_repo = registry_repos["${configname}_old"]
                  if (old_repo) {
                    pipeutils.tryWithOrWithoutCredentials([file(variable: 'OLD_REGISTRY_SECRET',
                                                                credentialsId: 'oscontainer-push-old-registry-secret')]) {
                      def authArg = "--authfile=\${REGISTRY_SECRET}"
                      if (env.OLD_REGISTRY_SECRET) {
                        authArg += " --dest-authfile=\${OLD_REGISTRY_SECRET}"
                      }
                      shwrap("""
                      cosa copy-container ${authArg} ${tag_args} \
                          ${repo} ${old_repo}
                      """)
                    }
                  }
              }
            }]}
          }
        }

        currentBuild.result = 'SUCCESS'

// main try finishes here
} catch (e) {
    currentBuild.result = 'FAILURE'
    throw e
} finally {
    if (official && currentBuild.result != 'SUCCESS') {
        slackSend(color: 'danger', message: ":fcos: :bullettrain_front: :trashfire: release <${env.BUILD_URL}|#${env.BUILD_NUMBER}> [${params.STREAM}][${params.ARCHES}] (${params.VERSION})")
    }
}}} // try-catch-finally, cosaPod and lock finish here
