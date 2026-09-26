import time
import pygame
from OpenGL.GL import *
from OpenGL.GL.shaders import compileProgram, compileShader


VERTEX_SHADER = """
#version 120

attribute vec2 aPosition;

void main() {
    gl_Position = vec4(aPosition, 0.0, 1.0);
}
"""


FRAGMENT_SHADER = """
#version 120

uniform vec2 iResolution;
uniform float iTime;
uniform float uHue;
uniform float uXOffset;
uniform float uSpeed;
uniform float uIntensity;
uniform float uSize;

#define OCTAVE_COUNT 10


vec3 hsv2rgb(vec3 c) {
    vec3 rgb = clamp(
        abs(mod(c.x * 6.0 + vec3(0.0, 4.0, 2.0), 6.0) - 3.0)
        - 1.0,
        0.0,
        1.0
    );

    return c.z * mix(vec3(1.0), rgb, c.y);
}


float hash11(float p) {
    p = fract(p * 0.1031);
    p *= p + 33.33;
    p *= p + p;
    return fract(p);
}


float hash12(vec2 p) {
    vec3 p3 = fract(vec3(p.xyx) * 0.1031);
    p3 += dot(p3, p3.yzx + 33.33);
    return fract((p3.x + p3.y) * p3.z);
}


mat2 rotate2d(float theta) {
    float c = cos(theta);
    float s = sin(theta);

    return mat2(
        c, -s,
        s,  c
    );
}


float noise(vec2 p) {
    vec2 ip = floor(p);
    vec2 fp = fract(p);

    float a = hash12(ip);
    float b = hash12(ip + vec2(1.0, 0.0));
    float c = hash12(ip + vec2(0.0, 1.0));
    float d = hash12(ip + vec2(1.0, 1.0));

    vec2 t = smoothstep(0.0, 1.0, fp);

    return mix(
        mix(a, b, t.x),
        mix(c, d, t.x),
        t.y
    );
}


float fbm(vec2 p) {
    float value = 0.0;
    float amplitude = 0.5;

    for (int i = 0; i < OCTAVE_COUNT; ++i) {
        value += amplitude * noise(p);

        p *= rotate2d(0.45);
        p *= 2.0;

        amplitude *= 0.5;
    }

    return value;
}


void main() {
    vec2 fragCoord = gl_FragCoord.xy;

    vec2 uv = fragCoord / iResolution.xy;

    uv = 2.0 * uv - 1.0;

    uv.x *= iResolution.x / iResolution.y;

    uv.x += uXOffset;

    uv += 2.0 * fbm(
        uv * uSize + 0.8 * iTime * uSpeed
    ) - 1.0;

    float dist = abs(uv.x);

    vec3 baseColor = hsv2rgb(
        vec3(
            uHue / 360.0,
            0.7,
            0.8
        )
    );

    vec3 col = baseColor *
        pow(
            mix(
                0.0,
                0.07,
                hash11(iTime * uSpeed)
            ) / dist,
            1.0
        ) *
        uIntensity;

    col = pow(col, vec3(1.0));

    float alpha = clamp(
        max(col.r, max(col.g, col.b)),
        0.0,
        1.0
    );

    gl_FragColor = vec4(col, alpha);
}
"""


def create_shader_program():
    vertex_shader = compileShader(
        VERTEX_SHADER,
        GL_VERTEX_SHADER
    )

    fragment_shader = compileShader(
        FRAGMENT_SHADER,
        GL_FRAGMENT_SHADER
    )

    return compileProgram(
        vertex_shader,
        fragment_shader
    )


def main():
    pygame.init()

    # Window: 1000x600
    pygame.display.set_mode(
        (1000, 600),
        pygame.OPENGL | pygame.DOUBLEBUF | pygame.RESIZABLE
    )

    pygame.display.set_caption("Python Lightning")

    # Transparantie zoals in de WebGL-versie
    glEnable(GL_BLEND)
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)

    program = create_shader_program()
    glUseProgram(program)

    # Full-screen quad
    vertices = [
        -1.0, -1.0,
         1.0, -1.0,
        -1.0,  1.0,

        -1.0,  1.0,
         1.0, -1.0,
         1.0,  1.0,
    ]

    vertex_buffer = glGenBuffers(1)

    glBindBuffer(GL_ARRAY_BUFFER, vertex_buffer)

    import ctypes

    vertex_data = (ctypes.c_float * len(vertices))(*vertices)

    glBufferData(
        GL_ARRAY_BUFFER,
        len(vertices) * 4,
        vertex_data,
        GL_STATIC_DRAW
    )

    position = glGetAttribLocation(
        program,
        "aPosition"
    )

    glEnableVertexAttribArray(position)

    glVertexAttribPointer(
        position,
        2,
        GL_FLOAT,
        GL_FALSE,
        0,
        None
    )

    # Uniform locations
    resolution_location = glGetUniformLocation(
        program,
        "iResolution"
    )

    time_location = glGetUniformLocation(
        program,
        "iTime"
    )

    hue_location = glGetUniformLocation(
        program,
        "uHue"
    )

    x_offset_location = glGetUniformLocation(
        program,
        "uXOffset"
    )

    speed_location = glGetUniformLocation(
        program,
        "uSpeed"
    )

    intensity_location = glGetUniformLocation(
        program,
        "uIntensity"
    )

    size_location = glGetUniformLocation(
        program,
        "uSize"
    )

    # Zelfde waarden als jouw React-component
    hue = 260.0
    x_offset = 0.0
    speed = 1.0
    intensity = 1.0
    size = 1.0

    start_time = time.perf_counter()

    clock = pygame.time.Clock()

    running = True

    while running:
        for event in pygame.event.get():

            if event.type == pygame.QUIT:
                running = False

            elif event.type == pygame.VIDEORESIZE:
                width = max(event.w, 1)
                height = max(event.h, 1)

                pygame.display.set_mode(
                    (width, height),
                    pygame.OPENGL |
                    pygame.DOUBLEBUF |
                    pygame.RESIZABLE
                )

        width, height = pygame.display.get_surface().get_size()

        glViewport(
            0,
            0,
            width,
            height
        )

        # Clear screen
        glClearColor(
            0.0,
            0.0,
            0.0,
            0.0
        )

        glClear(GL_COLOR_BUFFER_BIT)

        current_time = (
            time.perf_counter() - start_time
        )

        # Shader uniforms
        glUniform2f(
            resolution_location,
            width,
            height
        )

        glUniform1f(
            time_location,
            current_time
        )

        glUniform1f(
            hue_location,
            hue
        )

        glUniform1f(
            x_offset_location,
            x_offset
        )

        glUniform1f(
            speed_location,
            speed
        )

        glUniform1f(
            intensity_location,
            intensity
        )

        glUniform1f(
            size_location,
            size
        )

        # Render
        glDrawArrays(
            GL_TRIANGLES,
            0,
            6
        )

        pygame.display.flip()

        # ~60 FPS
        clock.tick(60)

    pygame.quit()


if __name__ == "__main__":
    main()
