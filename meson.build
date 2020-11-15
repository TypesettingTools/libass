project('libass', 'c', license: 'ISC',
        meson_version: '>= 0.55.0',
        default_options: ['c_std=gnu99', 'buildtype=debugoptimized', 'warning_level=2'],
        version: '0.15.0')

conf = configuration_data()
deps = []

if get_option('large-tiles')
    conf.set('CONFIG_LARGE_TILES', 1)
endif

cc = meson.get_compiler('c')
if cc.get_argument_syntax() == 'gcc'
    cflags = [
        '-Wno-sign-compare',
        '-Wno-unused-parameter',
        '-Werror-implicit-function-declaration',
        '-Wstrict-prototypes',
        '-Wpointer-arith',
        '-Wredundant-decls',
        '-Wno-missing-field-initializers'
    ]
    add_project_arguments(cc.get_supported_arguments(cflags), language: 'c')
endif
link_args = cc.get_supported_link_arguments(['-prefer-non-pic'])

check_functions = [
    'strdup',
    'strndup'
]

m_dep = cc.find_library('m', required: false)
if m_dep.found()
    deps += m_dep
endif

iconv_dep = cc.find_library('iconv', required: false)
if iconv_dep.found()
    deps += iconv_dep
endif
if iconv_dep.found() or cc.has_function('iconv_open')
    conf.set('CONFIG_ICONV', 1)
endif

# ASM
nasm_args = []
nasm = find_program('nasm', required: get_option('asm'))
enable_asm = nasm.found()

if enable_asm
    # NASM version check
    nasm_r = run_command(nasm, '-v')
    out = nasm_r.stdout().strip().split()
    if out[1].to_lower() == 'version'
        if out[2].version_compare('< 2.10')
            warning('nasm is too old (found @0@); ASM functions are disabled.'.format(out[2]))
            enable_asm = false
        endif
    else
        warning('Unexpected nasm version string: @0@; disabling.'.format(nasm_r.stdout()))
        enable_asm = false
    endif
endif

# Architecture check
if host_machine.cpu_family() == 'x86'
    bittype = '32'
    nasm_args += '-DARCH_X86_64=0'
    intel = true
elif host_machine.cpu_family() == 'x86_64'
    bittype = '64'
    nasm_args += ['-DARCH_X86_64=1', '-DPIC']
    intel = true
else
    intel = false
endif

if enable_asm and not intel
    warning('ASM functions are not available on non-intel machines; disabling.')
    enable_asm = false
elif (not enable_asm) and intel
    warning('Install nasm-2.10 or later for a significantly faster libass build.')
endif

if enable_asm
    conf.set('CONFIG_ASM', 1)

    if host_machine.system() == 'windows'
        nasm_args += ['-f', 'win' + bittype]
        if bittype == '64'
            nasm_args += ['-DHAVE_ALIGNED_STACK=1']
        else
            nasm_args += ['-DHAVE_ALIGNED_STACK=0', '-DPREFIX']
        endif
    elif host_machine.system() == 'darwin'
        nasm_args += ['-f', 'macho' + bittype, '-DPREFIX', '-DHAVE_ALIGNED_STACK=1']
    else
        nasm_args += ['-f', 'elf' + bittype, '-DHAVE_ALIGNED_STACK=1']
    endif

    nasm_args += ['-DHAVE_CPUNOP=0', '-Dprivate_prefix=ass']
    nasm_args += ['-I', meson.current_source_dir() / 'libass' + '/', '-o', '@OUTPUT@', '@INPUT@']

    nasm_gen = generator(nasm, output: '@BASENAME@.o', arguments: nasm_args)
endif

# Misc deps
freetype_dep = dependency('freetype', version: '>= 2.2.1',
                          fallback: ['freetype2', 'freetype2_dep'])
if freetype_dep.found()
    deps += freetype_dep
    conf.set('CONFIG_FREETYPE', 1)
endif

have_visibility_hidden = cc.has_argument('-fvisibility=hidden')
if have_visibility_hidden
    fribidi_visibility = '-DFRIBIDI_ENTRY=__attribute__ ((visibility ("default")))'
else
    if host_machine.system() == 'windows' and get_option('default_library') == 'static'
        fribidi_visibility = '-DFRIBIDI_LIB_STATIC'
    endif
endif
fribidi_dep = dependency('fribidi', version: '>= 0.19.0',
                         fallback: ['fribidi', 'libfribidi_dep'],
                         default_options: ['docs=false'])
if fribidi_dep.found()
    deps += fribidi_dep
    conf.set('CONFIG_FRIBIDI', 1)
endif

harfbuzz_dep = dependency('harfbuzz', version: '>= 0.9.5', required: get_option('harfbuzz'),
                          fallback: ['harfbuzz', 'libharfbuzz_dep'])
if harfbuzz_dep.found()
    deps += harfbuzz_dep
    conf.set('CONFIG_HARFBUZZ', 1)
endif

req_fontconfig = get_option('fontconfig')

if host_machine.system() != 'windows' and host_machine.system() != 'darwin'
    req_fontconfig = get_option('fontconfig').disabled()
endif

fontconfig_dep = dependency('fontconfig', version: '>= 2.10.92', required: req_fontconfig)
fontconfig = false
if fontconfig_dep.found()
    deps += fontconfig_dep
    conf.set('CONFIG_FONTCONFIG', 1)
    fontconfig = true
endif

if get_option('test')
    deps += dependency('libpng', version: '>= 1.2.0',
                       fallback: ['libpng', 'png_dep'])
    conf.set('CONFIG_LIBPNG', 1)
endif

# DirectWrite
directwrite = false
if host_machine.system() == 'windows' and not get_option('directwrite').disabled()
    if cc.has_header('windows.h')
        deps += cc.find_library('dwrite', required: true)
        conf.set('CONFIG_DIRECTWRITE', 1)
        directwrite = true
    elif get_option('directwrite').enabled()
        error('DirectWrite was enabled explicitly, but required header is missing.')
    endif
endif

# CoreText
coretext = false
if host_machine.system() == 'darwin' and not get_option('coretext').disabled()
    coretext_dep = dependency('appleframeworks', modules: ['CoreText', 'CoreFoundation'], required: false)
    if cc.has_type('CTRunRef', prefix: '#include <CoreText/CoreText.h>', dependencies: coretext_dep)
        deps += coretext_dep
        conf.set('CONFIG_CORETEXT', 1)
        coretext = true
    elif get_option('coretext').enabled()
        error('CoreText was enabled explicitly, but required headers or frameworks are missing.')
    endif
endif

if get_option('require-system-font-provider') and not fontconfig and not directwrite and not coretext
    error('''Either DirectWrite (on Windows), CoreText (on OSX), or Fontconfig (Linux, other) is required.
If you really want to compile without a system font provider, set -Drequire-system-font-provider=false''')
endif

foreach name: check_functions
    # workaround for https://github.com/mesonbuild/meson/issues/3672
    if cc.has_function(name) and cc.has_header_symbol('string.h', name)
        conf.set('HAVE_@0@'.format(name.to_upper()), 1)
    endif
endforeach

conf.set('PACKAGE_NAME', 'libass')
conf.set('PACKAGE_VERSION', meson.project_version())

config_h = configure_file(output: 'config.h', configuration: conf)

incs = include_directories('.', 'libass')

subdir('libass')
if get_option('test')
    subdir('test')
endif
if get_option('profile')
    subdir('profile')
endif

# libass.pc
pkg = import('pkgconfig')
pkg.generate(libass,
             name: 'libass',
             description: 'LibASS is an SSA/ASS subtitles rendering library',
             subdirs: meson.project_name())
