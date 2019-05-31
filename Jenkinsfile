def pod, utils, prod, devel_prefix, src_config_url, src_config_ref
node {
    checkout scm
    pod = readFile(file: "manifests/pod.yaml")
    utils = load("utils.groovy")

    // just autodetect if we're in prod or not
    def prod_jenkins = (env.JENKINS_URL == 'https://jenkins-fedora-coreos.apps.ci.centos.org/')
    def prod_job = (env.JOB_NAME == 'fedora-coreos/fedora-coreos-pipeline')
    prod = (prod_jenkins && prod_job)

    if (prod) {
        echo "Running in prod mode."
    } else {
        echo "Running in devel mode on ${env.JENKINS_URL}."
    }

    devel_prefix = utils.get_pipeline_annotation('dev-prefix')
    src_config_url = utils.get_pipeline_annotation('source-config-url')
    src_config_ref = utils.get_pipeline_annotation('source-config-ref')

    // sanity check that a valid prefix is provided if in devel mode and drop
    // the trailing '-' in the devel prefix
    if (!prod) {
      assert devel_prefix.length() > 0 : "Missing devel prefix"
      assert devel_prefix.endsWith("-") : "Missing trailing dash in devel prefix"
      devel_prefix = devel_prefix[0..-2]
    }
}

properties([
    disableConcurrentBuilds(),
    pipelineTriggers(prod ? [cron("H/30 * * * *")] : []),
    parameters([
      choice(name: 'STREAM',
             // XXX: Just pretend we're the testing stream for now... in
             // reality, we're closer to what "bodhi-updates" will be. Though the
             // testing stream is the main stream.
             choices: ['testing' /*, 'stable', 'testing-devel', 'bodhi-updates', etc... */ ],
             description: 'Fedora CoreOS stream to build',
             required: true)
    ])
])

def s3_builddir
if (prod) {
  s3_builddir = "fcos-builds/prod/streams/${params.STREAM}"
} else {
  // One prefix = one pipeline = one stream; the devel-up script is geared
  // towards testing a specific combination of (cosa, pipeline, fcos config),
  // not a full duplication of all the prod streams. One can always instantiate
  // a second prefix to test a separate combination if more than 1 concurrent
  // devel pipeline is needed.
  s3_builddir = "fcos-builds/devel/streams/${devel_prefix}"
}

podTemplate(cloud: 'openshift', label: 'coreos-assembler', yaml: pod, defaultContainer: 'jnlp') {
    node('coreos-assembler') { container('coreos-assembler') {

        stage('Init') {
            utils.shwrap("""
            # just always restart from scratch in case it's a devel pipeline
            # and it changed source url or ref
            rm -rf src/config

            # in the future, the stream will dictate the branch in the prod path
            coreos-assembler init --force --branch ${src_config_ref} ${src_config_url}
            """)
        }

        stage('Fetch') {
            /*
            // XXX: uncomment once we have a build there
            if (utils.path_exists("/.aws")) {
                utils.shwrap("""
                coreos-assembler buildprep s3://${s3_builddir}
                """)
            }
            */

            if (prod) {
                // make sure our cached version matches prod exactly before continuing
                utils.rsync_in("builds", "builds")
            }
        }

        def prevBuildID = null
        if (utils.path_exists("builds/latest")) {
            prevBuildID = utils.shwrap_capture("readlink builds/latest")
        }

        stage('Build') {
            utils.shwrap("""
            coreos-assembler build --skip-prune
            """)
        }

        def newBuildID = utils.shwrap_capture("readlink builds/latest")
        if (prevBuildID == newBuildID) {
            currentBuild.result = 'SUCCESS'
            currentBuild.description = "💤 (no new build)"
            return
        } else {
            currentBuild.description = "⚡ ${newBuildID}"
        }

        stage('Build Metal') {
            utils.shwrap("""
            coreos-assembler buildextend-metal
            """)
        }

        stage('Build Installer') {
            utils.shwrap("""
            coreos-assembler buildextend-installer
            """)
        }

        stage('Build Openstack') {
            utils.shwrap("""
            coreos-assembler buildextend-openstack
            """)
        }

        stage('Build VMware') {
            utils.shwrap("""
            coreos-assembler buildextend-vmware
            """)
        }

        stage('Prune') {
            // XXX: stop pruning like this when we fully drop artifact server
            utils.shwrap("""
            coreos-assembler prune --keep=8
            """)

            // If the cache img is larger than e.g. 8G, then nuke it. Otherwise
            // it'll just keep growing and we'll hit ENOSPC.
            utils.shwrap("""
            if [ \$(du cache/cache.qcow2 | cut -f1) -gt \$((1024*1024*8)) ]; then
                rm -vf cache/cache.qcow2
                qemu-img create -f qcow2 cache/cache.qcow2 10G
                LIBGUESTFS_BACKEND=direct virt-format --filesystem=xfs -a cache/cache.qcow2
            fi
            """)
        }

        stage('Archive') {

            // First, compress image artifacts
            utils.shwrap("""
            coreos-assembler compress
            """)

            // Change perms to allow reading on webserver side.
            // Don't touch symlinks (https://github.com/CentOS/sig-atomic-buildscripts/pull/355)
            // XXX: can drop this when dropping artifact server
            utils.shwrap("""
            find builds/ ! -type l -exec chmod a+rX {} +
            """)

            // Note that if the prod directory doesn't exist on the remote this
            // will fail. We can possibly hack around this in the future:
            // https://stackoverflow.com/questions/1636889
            if (prod) {
                utils.rsync_out("builds", "builds")
            }

            if (utils.path_exists("/.aws")) {
              // XXX: just upload as public-read for now
              utils.shwrap("""
              coreos-assembler upload s3 --acl=public-read ${s3_builddir}
              """)
            }
        }
    }}
}
